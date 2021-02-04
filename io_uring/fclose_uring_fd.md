what happened when we fclose(uring_fd)?
We know that one uring_fd associate with one ctx. So close a uring_fd means clean a io_uring ctx.
Let's first look into filp_close() and go from there

#filp_close()
```c
1270 int filp_close(struct file *filp, fl_owner_t id)
1271 {
1272         int retval = 0;
1273
1274         if (!file_count(filp)) {
1275                 printk(KERN_ERR "VFS: Close: file count is 0\n");
1276                 return 0;
1277         }
1278
1279         if (filp->f_op->flush)
1280                 retval = filp->f_op->flush(filp, id);
1281
1282         if (likely(!(filp->f_mode & FMODE_PATH))) {
1283                 dnotify_flush(filp, id);
1284                 locks_remove_posix(filp, id);
1285         }
1286         fput(filp);
1287         return retval;
1288 }
```
we can see it mainly calls filp->f_op->flush() and then fput(filp)
for uring_fd, f_op->flush = io_uring_flush, let's go into it

#io_uring_flush()
```c
9142 static int io_uring_flush(struct file *file, void *data)
9143 {
9144         struct io_uring_task *tctx = current->io_uring;
9145         struct io_ring_ctx *ctx = file->private_data;
9146
		// [Hao Xu] cancel all requests if we are exiting the thread
9147         if (fatal_signal_pending(current) || (current->flags & PF_EXITING))
9148                 io_uring_cancel_task_requests(ctx, NULL);
9149
9150         if (!tctx)
9151                 return 0;
9152
9153         /* we should have cancelled and erased it before PF_EXITING */
9154         WARN_ON_ONCE((current->flags & PF_EXITING) &&
9155                 ¦    xa_load(&tctx->xa, (unsigned long)file));
9156
9157         /*
9158         ¦* fput() is pending, will be 2 if the only other ref is our potential
9159         ¦* task file note. If the task is exiting, drop regardless of count.
9160         ¦*/
9161         if (atomic_long_read(&file->f_count) != 2)
9162                 return 0;
9163
9164         if (ctx->flags & IORING_SETUP_SQPOLL) {
9165                 /* there is only one file note, which is owned by sqo_task */
9166                 WARN_ON_ONCE(ctx->sqo_task != current &&
9167                         ¦    xa_load(&tctx->xa, (unsigned long)file));
9168                 /* sqo_dead check is for when this happens after cancellation */
9169                 WARN_ON_ONCE(ctx->sqo_task == current && !ctx->sqo_dead &&
9170                         ¦    !xa_load(&tctx->xa, (unsigned long)file));
9171
9172                 io_disable_sqo_submit(ctx);
9173         }
9174
		// [Hao Xu] watch out here, we don't erase file from current->io_uring
		// if sqpoll enabled and current is not the one which created this ctx(file)
9175         if (!(ctx->flags & IORING_SETUP_SQPOLL) || ctx->sqo_task == current)
9176                 io_uring_del_task_file(file);
9177         return 0;
9178 }
```
It does 4 things:
	- if task is exiting, quit all requests
	- check if file->f_count is 2
	- if sqpoll enabled, stop submiting new reqs in that sqthread
	- if sqpoll disabled or ctx is created by this task, delete uring file from current->io_uring->xa tree

it puzzles me a lot of the last point above: the task to close the uring file can be different with the one created
the uring file(which is ctx->sqo_task), then I think it doesn't make sense to delete uring file in the current->io_uring.

#io_uring_release()
We can see in io_uring_flush(), f_count should be 2, we do fput() in io_uring_del_task_file() and after we returns to filp_close()
we do fput() again, so now f_count is 0. What happend then?

graph TB
    id1(fput)-->id2(fput_many)--"filp->f_count is 0"-->id3(____fput)-->id4(__fput)-->id5(filp->f_op->release)-->id6(io_uring_release);

```c
8815 static int io_uring_release(struct inode *inode, struct file *file)
8816 {
8817         struct io_ring_ctx *ctx = file->private_data;
8818
8819         file->private_data = NULL;
8820         io_ring_ctx_wait_and_kill(ctx);
8821         return 0;
8822 }
```
It basically detachs uring file and its ctx, and then calls io_ring_ctx_wait_and_kill()

#io_ring_ctx_wait_and_kill()
```c
8773 static void io_ring_ctx_wait_and_kill(struct io_ring_ctx *ctx)
8774 {
8775         mutex_lock(&ctx->uring_lock);
8776         percpu_ref_kill(&ctx->refs);
8777
		// [Hao Xu] ctx->sqo_dead means we cannot submit sqes to sqthread any more
8778         if (WARN_ON_ONCE((ctx->flags & IORING_SETUP_SQPOLL) && !ctx->sqo_dead))
8779                 ctx->sqo_dead = 1;
8780
		// [Hao Xu] flush cqring overflow list forcely
8781         /* if force is set, the ring is going away. always drop after that */
8782         ctx->cq_overflow_flushed = 1;
8783         if (ctx->rings)
8784                 __io_cqring_overflow_flush(ctx, true, NULL, NULL);
8785         mutex_unlock(&ctx->uring_lock);
8786
8787         io_kill_timeouts(ctx, NULL, NULL);
8788         io_poll_remove_all(ctx, NULL, NULL);
8789
8790         if (ctx->io_wq)
8791                 io_wq_cancel_cb(ctx->io_wq, io_cancel_ctx_cb, ctx, true);
8792
8793         /* if we failed setting up the ctx, we might not have any rings */
8794         io_iopoll_try_reap_events(ctx);
8795         idr_for_each(&ctx->personality_idr, io_remove_personalities, ctx);
8796
8797         /*
8798         ¦* Do this upfront, so we won't have a grace period where the ring
8799         ¦* is closed but resources aren't reaped yet. This can cause
8800         ¦* spurious failure in setting up a new ring.
8801         ¦*/
8802         io_unaccount_mem(ctx, ring_pages(ctx->sq_entries, ctx->cq_entries),
8803                         ¦ACCT_LOCKED);
8804
8805         INIT_WORK(&ctx->exit_work, io_ring_exit_work);
8806         /*
8807         ¦* Use system_unbound_wq to avoid spawning tons of event kworkers
8808         ¦* if we're exiting a ton of rings at the same time. It just adds
8809         ¦* noise and overhead, there's no discernable change in runtime
8810         ¦* over using system_wq.
8811         ¦*/
8812         queue_work(system_unbound_wq, &ctx->exit_work);
8813 }
```

#io_ring_exit_work()
```c
8749 static void io_ring_exit_work(struct work_struct *work)
8750 {
8751         struct io_ring_ctx *ctx = container_of(work, struct io_ring_ctx,
8752                                         ¦      exit_work);
8753
8754         /*
8755         ¦* If we're doing polled IO and end up having requests being
8756         ¦* submitted async (out-of-line), then completions can come in while
8757         ¦* we're waiting for refs to drop. We need to reap these manually,
8758         ¦* as nobody else will be looking for them.
8759         ¦*/
8760         do {
8761                 __io_uring_cancel_task_requests(ctx, NULL);
8762         } while (!wait_for_completion_timeout(&ctx->ref_comp, HZ/20));
8763         io_ring_ctx_free(ctx);
8764 }
```

#io_ring_ctx_free()
```c
8668 static void io_ring_ctx_free(struct io_ring_ctx *ctx)
8669 {
8670         io_finish_async(ctx);
8671         io_sqe_buffer_unregister(ctx);
8672
8673         if (ctx->sqo_task) {
8674                 put_task_struct(ctx->sqo_task);
8675                 ctx->sqo_task = NULL;
8676                 mmdrop(ctx->mm_account);
8677                 ctx->mm_account = NULL;
8678         }
8679
8680 #ifdef CONFIG_BLK_CGROUP
8681         if (ctx->sqo_blkcg_css)
8682                 css_put(ctx->sqo_blkcg_css);
8683 #endif
8684
8685         io_sqe_files_unregister(ctx);
8686         io_eventfd_unregister(ctx);
8687         io_destroy_buffers(ctx);
8688         idr_destroy(&ctx->personality_idr);
8689
8690 #if defined(CONFIG_UNIX)
8691         if (ctx->ring_sock) {
8692                 ctx->ring_sock->file = NULL; /* so that iput() is called */
8693                 sock_release(ctx->ring_sock);
8694         }
8695 #endif
8696
8697         io_mem_free(ctx->rings);
8698         io_mem_free(ctx->sq_sqes);
8699
8700         percpu_ref_exit(&ctx->refs);
8701         free_uid(ctx->user);
8702         put_cred(ctx->creds);
8703         kfree(ctx->cancel_hash);
8704         kmem_cache_free(req_cachep, ctx->fallback_req);
8705         kfree(ctx);
8706 }
```

a graph including all:

graph TB
	id0(filp_close)-->id1(fput)-->id2(fput_many)--"filp->f_count is 0"-->id3(____fput)-->id4(__fput)-->id5(filp->f_op->release)-->id6(io_uring_release)-->id7(io_ring_ctx_wait_and_kill)-->id8(io_ring_exit_work)-->id9(io_ring_ctx_free)


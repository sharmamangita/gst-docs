# Sink elements

Sink elements consume data and normally have no source pads.

Typical sink elements include:

  - audio/video renderers

  - network sinks

  - filesinks

Sinks are harder to construct than other element types as they are
treated specially by the GStreamer core.

## state changes

A sink always returns `ASYNC` from the state change to `PAUSED`, this
includes a state change from `READY→PAUSED` and `PLAYING→PAUSED`. The reason
for this is that this way we can detect when the first buffer or event
arrives in the sink when the state change completes.

A sink should block on the first `EOS` event or buffer received in the
`READY→PAUSED` state before committing the state to `PAUSED`.

`FLUSHING` events have to be handled out of sync with the buffer flow and
take no part in the preroll procedure.

Events other than `EOS` do not complete the preroll stage.

## sink overview

  - TODO: `PREROLL_LOCK` can be removed and we can safely use the `STREAM_LOCK`.

```c
  # Commit the state. We return TRUE if we can continue
  # streaming, FALSE in the case we go to a READY or NULL state.
  # if we go to PLAYING, we don't need to block on preroll.
  commit
  {
    LOCK
    switch (pending)
      case PLAYING:
        need_preroll = FALSE
        break
      case PAUSED:
        break
      case READY:
      case NULL:
        return FALSE
      case VOID:
        return TRUE

    # update state
    state = pending
    next = VOID
    pending = VOID
    UNLOCK
    return TRUE
  }

  # Sync an object. We have to wait for the element to reach
  # the PLAYING state before we can wait on the clock.
  # Some items do not need synchronisation (most events) so the
  # get_times method returns FALSE (not syncable)
  # need_preroll indicates that we are not in the PLAYING state
  # and therefore need to commit and potentially block on preroll
  # if our clock_wait got interrupted we commit and block again.
  # The reason for this is that the current item being rendered is
  # not yet finished and we can use that item to finish preroll.
  do_sync (obj)
  {
    # get timing information for this object
    syncable = get_times (obj, &start, &stop)
    if (!syncable)
      return OK;
 again:
    while (need_preroll)
      if (need_commit)
        need_commit = FALSE
        if (!commit)
          return FLUSHING

      if (need_preroll)
        # release PREROLL_LOCK and wait. prerolled can be observed
        # and will be TRUE
        prerolled = TRUE
        PREROLL_WAIT (releasing PREROLL_LOCK)
        prerolled = FALSE
        if (flushing)
          return FLUSHING

    if (valid (start || stop))
      PREROLL_UNLOCK
      end_time = stop
      ret = wait_clock (obj,start)
      PREROLL_LOCK
      if (flushing)
        return FLUSHING
      # if the clock was unscheduled, we redo the
      # preroll
      if (ret == UNSCHEDULED)
        goto again
  }

  # render a prerollable item (EOS or buffer). It is
  # always called with the PREROLL_LOCK helt.
  render_object (obj)
  {
    ret = do_sync (obj)
    if (ret != OK)
      return ret;

    # preroll and syncing done, now we can render
    render(obj)
  }
                                   | # sinks that sync on buffer contents do like this
                                   | while (more_to_render)
                                   |   ret = render
                                   |   if (ret == interrupted)
                                   |     prerolled = TRUE
    render (buffer)          ----->|     PREROLL_WAIT (releasing PREROLL_LOCK)
                                   |     prerolled = FALSE
                                   |     if (flushing)
                                   |       return FLUSHING
                                   |

  # queue a prerollable item (EOS or buffer). It is
  # always called with the PREROLL_LOCK helt.
  # This function will commit the state when receiving the
  # first prerollable item.
  # items are then added to the rendering queue or rendered
  # right away if no preroll is needed.
  queue (obj, prerollable)
  {
    if (need_preroll)
      if (prerollable)
        queuelen++

      # first item in the queue while we need preroll
      # will complete state change and call preroll
      if (queuelen == 1)
        preroll (obj)
        if (need_commit)
          need_commit = FALSE
          if (!commit)
            return FLUSHING

      # then see if we need more preroll items before we
      # can block
      if (need_preroll)
        if (queuelen <= maxqueue)
          queue.add (obj)
          return OK

    # now clear the queue and render each item before
    # rendering the current item.
    while (queue.hasItem)
      render_object (queue.remove())

    render_object (obj)
    queuelen = 0
  }

  # various event functions
  event
    EOS:
      # events must complete preroll too
      STREAM_LOCK
      PREROLL_LOCK
      if (flushing)
        return FALSE
      ret = queue (event, TRUE)
      if (ret == FLUSHING)
        return FALSE
      PREROLL_UNLOCK
      STREAM_UNLOCK
      break
    SEGMENT:
      # the segment must be used to clip incoming
      # buffers. Then go into the queue as non-prerollable
      # items used for syncing the buffers
      STREAM_LOCK
      PREROLL_LOCK
      if (flushing)
        return FALSE
      set_clip
      ret = queue (event, FALSE)
      if (ret == FLUSHING)
        return FALSE
      PREROLL_UNLOCK
      STREAM_UNLOCK
      break
    FLUSH_START:
      # set flushing and unblock all that is waiting
      event                   ----> subclasses can interrupt render
      PREROLL_LOCK
      flushing = TRUE
      unlock_clock
      PREROLL_SIGNAL
      PREROLL_UNLOCK
      STREAM_LOCK
      lost_state
      STREAM_UNLOCK
      break
    FLUSH_END:
      # unset flushing and clear all data and eos
      STREAM_LOCK
      event
      PREROLL_LOCK
      queue.clear
      queuelen = 0
      flushing = FALSE
      eos = FALSE
      PREROLL_UNLOCK
      STREAM_UNLOCK
      break

  # the chain function checks the buffer falls within the
  # configured segment and queues the buffer for preroll and
  # rendering
  chain
    STREAM_LOCK
    PREROLL_LOCK
    if (flushing)
      return FLUSHING
    if (clip)
      queue (buffer, TRUE)
    PREROLL_UNLOCK
    STREAM_UNLOCK

  state
    switch (transition)
      READY_PAUSED:
        # no datapassing is going on so we always return ASYNC
        ret = ASYNC
        need_commit = TRUE
        eos = FALSE
        flushing = FALSE
        need_preroll = TRUE
        prerolled = FALSE
        break
      PAUSED_PLAYING:
        # we grab the preroll lock. This we can only do if the
        # chain function is either doing some clock sync, we are
        # waiting for preroll or the chain function is not being called.
        PREROLL_LOCK
        if (prerolled || eos)
          ret = OK
          need_commit = FALSE
          need_preroll = FALSE
          if (eos)
            post_eos
          else
            PREROLL_SIGNAL
        else
          need_preroll = TRUE
          need_commit = TRUE
          ret = ASYNC
        PREROLL_UNLOCK
        break
      PLAYING_PAUSED:
                           ---> subclass can interrupt render
        # we grab the preroll lock. This we can only do if the
        # chain function is either doing some clock sync
        # or the chain function is not being called.
        PREROLL_LOCK
        need_preroll = TRUE
        unlock_clock
        if (prerolled || eos)
          ret = OK
        else
          ret = ASYNC
        PREROLL_UNLOCK
        break
      PAUSED_READY:
                           ---> subclass can interrupt render
        # we grab the preroll lock. Set to flushing and unlock
        # everything. This should exit the chain functions and stop
        # streaming.
        PREROLL_LOCK
        flushing = TRUE
        unlock_clock
        queue.clear
        queuelen = 0
        PREROLL_SIGNAL
        ret = OK
        PREROLL_UNLOCK
        break
```

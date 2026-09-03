# QEMU Virtio & Vhost Subsystem Guide

Virtio (`hw/virtio/`, `include/hw/virtio/`) is the standardized paravirtualized I/O framework in QEMU, supporting network (virtio-net), block (virtio-blk), console, GPU, and SCSI devices.

---

## 1. VirtQueue Processing Lifecycle

Virtqueues connect guest memory ring buffers to host device models:

```
Guest driver populates descriptors -> Guest kicks virtqueue -> Host pops element -> Host processes I/O -> Host pushes element -> Host injects IRQ
```

### 1.1 Popping and Pushing Elements

```c
VirtQueueElement *elem;

while ((elem = virtqueue_pop(vq, sizeof(VirtQueueElement)))) {
    /* 1. Process guest-provided buffers */
    size_t len = process_request(s, elem->in_sg, elem->in_num,
                                 elem->out_sg, elem->out_num);

    /* 2. Complete and push element back to ring */
    virtqueue_push(vq, elem, len);
    g_free(elem);
}

/* 3. Notify guest of completions */
virtio_notify(vdev, vq);
```

### 1.2 Element Leak Invariant

- Every `VirtQueueElement` returned by `virtqueue_pop()` **MUST** be:
  - Either completed with `virtqueue_push(vq, elem, len)` and freed with `g_free(elem)`.
  - Or detached with `virtqueue_detach_element(vq, elem, len)` and freed if the request was canceled or abandoned.
- Dropping an element without pushing or detaching it corrupts the virtqueue ring in the guest.

---

## 2. Security & Untrusted Ring Descriptors

The guest controls the virtqueue descriptor table, available ring, and used ring:

1. **Scatter-Gather List Limits**:
   - Verify `elem->in_num` and `elem->out_num` do not exceed expected limits (`VIRTQUEUE_MAX_SIZE`, typically 1024).
2. **Buffer Bounds**:
   - Do not trust total buffer size calculated from guest scatter-gather vectors without validating against request headers.
3. **Descriptor Loops**:
   - `virtqueue_pop()` internally detects circular descriptor references. Always handle `elem == NULL` gracefully as an empty or corrupted queue.

---

## 3. Vhost & Vhost-User Invariants

Vhost offloads virtqueue processing to the host kernel (`vhost-kernel`) or an external userspace process (`vhost-user`).

### 3.1 Vhost Lifecycle
- `vhost_dev_init()` / `vhost_dev_cleanup()`
- `vhost_dev_start()` / `vhost_dev_stop()`
- **Invariant**: The device model must handle vhost-user socket disconnections gracefully at any point without crashing QEMU.

### 3.2 Feature Negotiation
- Devices define host features via `get_features`.
- Validate that features requiring backend support (e.g. `VIRTIO_F_VERSION_1`, `VIRTIO_RING_F_EVENT_IDX`) are properly masked if the backend lacks support.

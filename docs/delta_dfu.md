# Delta DFU

Delta DFU lets MCUboot accept a signed patch image in the secondary slot and
apply it to the current primary image. The patch image is a normal MCUboot image
with the `IMAGE_F_DELTA` header flag set. It is transport-neutral: any DFU
transport that can write the patch image to the secondary slot and mark it
pending can use it.

Delta images are supported by the overwrite-only boot path. They are not
compressed images and they do not use the decompression path.

## Image Format

The delta image payload starts with this little-endian header:

```c
struct boot_delta_header {
    uint32_t magic;       /* "MDL1" */
    uint16_t version;     /* 1, or 2 for reversible deltas */
    uint16_t header_size; /* sizeof(struct boot_delta_header) */
    uint32_t target_size; /* reconstructed signed target image size */
    uint32_t write_size;  /* primary-slot span covered by the patch */
    uint32_t record_count;
    uint32_t block_size;
    uint32_t flags;       /* present when version is 2 */
    uint32_t base_size;   /* present when version is 2 */
};
```

The header is followed by `record_count` records:

```c
struct boot_delta_record {
    uint32_t offset;
    uint32_t size;
    uint8_t  new_data[size];
    uint8_t  old_data[size]; /* present when BOOT_DELTA_F_RESTORE is set */
};
```

Each record replaces `size` bytes at `offset` in the primary slot. Records are
strictly ordered and non-overlapping, and record data is padded to 4 bytes in
the patch payload.

Version 1 records contain only `new_data`. Version 2 records can set
`BOOT_DELTA_F_RESTORE`; in that case each record also contains the old bytes
from the base image. The additional old bytes let MCUboot apply the same signed
delta image in the reverse direction.

The patch image contains protected TLVs with the hash of the expected base
image and the hash of the reconstructed target image:

- `IMAGE_TLV_DELTA_BASE_SHA`
- `IMAGE_TLV_DELTA_TARGET_SHA`

During boot, MCUboot validates the signed delta image, checks that the active
primary image hash matches `IMAGE_TLV_DELTA_BASE_SHA`, applies the records to
the primary slot, validates the reconstructed target image, and compares its
hash with `IMAGE_TLV_DELTA_TARGET_SHA`.

## Zephyr Configuration

Enable delta DFU in the Zephyr MCUboot image with:

```text
CONFIG_BOOT_UPGRADE_ONLY=y
CONFIG_BOOT_DELTA_DFU=y
```

To allow unconfirmed test delta updates to restore the previous image, also
enable:

```text
CONFIG_BOOT_DELTA_DFU_RESTORE=y
```

For sysbuild, select overwrite-only mode at the sysbuild level:

```text
SB_CONFIG_MCUBOOT_MODE_OVERWRITE_ONLY=y
```

On flash devices that require erase before write, MCUboot preserves unchanged
sector bytes in a RAM buffer while applying delta records. Configure the buffer
with `CONFIG_BOOT_DELTA_DFU_SECTOR_BUFFER_SIZE`; it must be at least as large as
the largest primary-slot erase sector touched by a delta update.

## Creating Delta Images

Create the current signed base image and the full signed target image normally.
Then create the delta image by passing the signed base image to `imgtool sign`:

```bash
./scripts/imgtool.py sign \
  --version 1.0.1+0 \
  --header-size 0x800 \
  --slot-size 729088 \
  --overwrite-only \
  --align 1 \
  --key root-rsa-2048.pem \
  --delta-base app-base.signed.bin \
  --delta-block-size 16 \
  --delta-revertible \
  app-target.bin \
  app-target.delta.signed.bin
```

`--delta-block-size` controls the comparison granularity. It must be a positive
power-of-two multiple of 4 and should be a multiple of the target flash write
alignment.

`--delta-revertible` is optional. It increases the delta payload by storing the
old bytes for each changed record, but it is still usually much smaller than
keeping a full second image slot.

## Restore Behavior

Reversible delta restore uses the same secondary slot that received the update;
it does not require a transport-specific path. When a reversible delta image is
marked as a test update, MCUboot applies the forward records, validates the new
primary image, then erases only the secondary-slot trailer and writes primary
trailer state for a pending restore. The signed delta payload remains in the
secondary slot.

If the new application confirms itself, the primary trailer `image_ok` flag is
set and the restore is cancelled. If the device reboots before confirmation,
MCUboot sees the primary trailer state, validates the same signed delta image in
the secondary slot, applies the old bytes from each record, validates the
restored image against `IMAGE_TLV_DELTA_BASE_SHA`, and clears the restore state.

- bootutil: added overwrite-only delta DFU support with signed patch images.
- bootutil: added optional reversible delta restore for unconfirmed test
  updates.
- imgtool: added `--delta-base` and `--delta-block-size` for generating signed
  delta images.
- imgtool: added `--delta-revertible` for generating signed deltas that carry
  the old bytes needed for restore.

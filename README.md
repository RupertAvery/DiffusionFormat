# Diffusion Format

Repository for specs and scripts for a container format for diffusion model checkpoints

# Motivation

A diffusion model is a file that contains the weights resulting from training, or fine-tuning an existing model for text-to-image applications.

Diffusion models are currently distributed in the following formats:

* .ckpt
* .safetensors

When a text-to-image application generates an image, the generation parameters are embedded in the image as metadata, one of data embedded in the model hash.

The model hash is used to uniquely identify the source model that was used to generate the image.

The current implementation of calculating this hash is given as the SHA256 of the first 0x10000 bytes at offset 0x100000.  This is done as the model is loaded.

```py
def model_hash(filename):
    try:
        with open(filename, "rb") as file:
            import hashlib
            m = hashlib.sha256()

            file.seek(0x100000)
            m.update(file.read(0x10000))
            return m.hexdigest()[0:8]
    except FileNotFoundError:
        return 'NOFILE'
```

Unfortunately, this method is prone to collisions, as there is no guarantee that the given set of bytes will be different from one model to another, especially ones 
that are fine-tuned from the same base model, or checkpoints used to continue training.

We need a better way to calculate hashes, one that is not necessarily calculated in real time.  Additionally, we would like to incorprate metadata with the model,
such as author info, description, notes and trigger words used in training.

# Proposal

We propose a container format for diffusion checkpoints to support:

* a proper complete file hash, precalculated and stored in the header
* metadata, easily editable, that does not affect the model hash.

The container consists of a header, the metadata, and the model, raw, as .ckpt is already compressed. 

The header must ensure that there is enough space allocated to store the metadata, as well as allow any edits that authors may want to perform, without needing
to move the model.

# Proposed Format

| Offset     | Size     | Description                        |
|------------|----------|------------------------------------|
| 0000       | 4 B      | 'D''F''S''N' - Magic               |
| 0004       | 2 B      | Type  01 - CKPT, 02 - Saftensors   |
| 0006       | 4 B      | Offset to metadata from BOF        |
| 000A       | 4 B      | Offset to model from BOF           |
| 1000       | 2 MB     | Metadata                           |
| ????       | ? GB     | Model                              |


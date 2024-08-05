Sure, here's a draft for your README:

````markdown
# Flux FP16 Accumulate Model Implementation with FastAPI

This repository contains an implementation of the Flux model, along with an API that allows you to generate images based on text prompts. The API can be run via command-line arguments.

## Table of Contents

-   [Installation](#installation)
-   [Usage](#usage)
-   [Configuration](#configuration)
-   [API Endpoints](#api-endpoints)
-   [Examples](#examples)
-   [License](#license)

## Installation

To install the required dependencies, run:

```bash
pip install -r requirements.txt
```
````

## Usage

You can run the API server using the following command:

```bash
python main.py --config-path <path_to_config> --port <port_number> --host <host_address>
```

### Command-Line Arguments

-   `--config-path`: Path to the configuration file. If not provided, the model will be loaded from the command line arguments.
-   `--port`: Port to run the server on (default: 8088).
-   `--host`: Host to run the server on (default: 0.0.0.0).
-   `--flow-model-path`: Path to the flow model.
-   `--text-enc-path`: Path to the text encoder.
-   `--autoencoder-path`: Path to the autoencoder.
-   `--model-version`: Choose model version (`flux-dev` or `flux-schnell`).
-   `--flux-device`: Device to run the flow model on (default: cuda:0).
-   `--text-enc-device`: Device to run the text encoder on (default: cuda:0).
-   `--autoencoder-device`: Device to run the autoencoder on (default: cuda:0).
-   `--num-to-quant`: Number of linear layers in the flow transformer to quantize (default: 20).

## Configuration

The configuration files are located in the `configs` directory. You can specify different configurations for different model versions and devices.

Example configuration file (`configs/config-dev.json`):

```json
{
    "version": "flux-dev",
    "params": {
        "in_channels": 64,
        "vec_in_dim": 768,
        "context_in_dim": 4096,
        "hidden_size": 3072,
        "mlp_ratio": 4.0,
        "num_heads": 24,
        "depth": 19,
        "depth_single_blocks": 38,
        "axes_dim": [16, 56, 56],
        "theta": 10000,
        "qkv_bias": true,
        "guidance_embed": true
    },
    "ae_params": {
        "resolution": 256,
        "in_channels": 3,
        "ch": 128,
        "out_ch": 3,
        "ch_mult": [1, 2, 4, 4],
        "num_res_blocks": 2,
        "z_channels": 16,
        "scale_factor": 0.3611,
        "shift_factor": 0.1159
    },
    "ckpt_path": "/path/to/your/flux1-dev.sft",
    "ae_path": "/path/to/your/ae.sft",
    "repo_id": "black-forest-labs/FLUX.1-dev",
    "repo_flow": "flux1-dev.sft",
    "repo_ae": "ae.sft",
    "text_enc_max_length": 512,
    "text_enc_path": "path/to/your/t5-v1_1-xxl-encoder-bf16", // or "city96/t5-v1_1-xxl-encoder-bf16" for a simple to download version
    "text_enc_device": "cuda:1",
    "ae_device": "cuda:1",
    "flux_device": "cuda:0",
    "flow_dtype": "float16",
    "ae_dtype": "bfloat16",
    "text_enc_dtype": "bfloat16",
    "num_to_quant": 20
}
```

## API Endpoints

### Generate Image

-   **URL**: `/generate`
-   **Method**: `POST`
-   **Request Body**:

    -   `prompt` (str): The text prompt for image generation.
    -   `width` (int, optional): The width of the generated image (default: 720).
    -   `height` (int, optional): The height of the generated image (default: 1024).
    -   `num_steps` (int, optional): The number of steps for the generation process (default: 24).
    -   `guidance` (float, optional): The guidance scale for the generation process (default: 3.5).
    -   `seed` (int, optional): The seed for random number generation.

-   **Response**: A JPEG image stream.

## Examples

### Running the Server

```bash
python main.py --config-path configs/config-dev.json --port 8088 --host 0.0.0.0
```

OR, if you need more granular control over the server, you can run the server with something like this:

```bash
python main.py --port 8088 --host 0.0.0.0 \
    --flow-model-path /path/to/your/flux1-dev.sft \
    --text-enc-path /path/to/your/t5-v1_1-xxl-encoder-bf16 \
    --autoencoder-path /path/to/your/ae.sft \
    --model-version flux-dev \
    --flux-device cuda:0 \
    --text-enc-device cuda:1 \
    --autoencoder-device cuda:1 \
    --num-to-quant 20
```

### Generating an Image

Send a POST request to `http://<host>:<port>/generate` with the following JSON body:

```json
{
    "prompt": "a beautiful asian woman in traditional clothing with golden hairpin and blue eyes, wearing a red kimono with dragon patterns",
    "width": 1024,
    "height": 1024,
    "num_steps": 24,
    "guidance": 3.0,
    "seed": 13456
}
```

For an example of how to generate from a python client using the FastAPI server:

```py
import requests
import io

prompt = "a beautiful asian woman in traditional clothing with golden hairpin and blue eyes, wearing a red kimono with dragon patterns"
res = requests.post(
    "http://localhost:8088/generate",
    json={
        "width": 1024,
        "height": 720,
        "num_steps": 20,
        "guidance": 4,
        "prompt": prompt,
    },
    stream=True,
)

with open(f"output.jpg", "wb") as f:
    f.write(io.BytesIO(res.content).read())

```

## License

This project is licensed under the MIT License.

````

## References

- Code for loading the pipeline from the configuration path:

```200:310:flux_impl.py
@torch.inference_mode()
def load_pipeline_from_config(config: ModelSpec) -> Model:
    models = load_models_from_config(config)
    config = models.config
    num_quanted = 0
    max_quanted = config.num_to_quant
    flux_device = into_device(config.flux_device)
    ae_device = into_device(config.ae_device)
    clip_device = into_device(config.text_enc_device)
    t5_device = into_device(config.text_enc_device)
    flux_dtype = into_dtype(config.flow_dtype)
    device_index = flux_device.index or 0
    flow_model = models.flow.requires_grad_(False).eval().type(flux_dtype)
    for block in flow_model.single_blocks:
        block.cuda(flux_device)
        if num_quanted < max_quanted:
            num_quanted = quant_module(
                block.linear1, num_quanted, device_index=device_index
            )

    for block in flow_model.double_blocks:
        block.cuda(flux_device)
        if num_quanted < max_quanted:
            num_quanted = full_quant(
                block, max_quanted, num_quanted, device_index=device_index
            )

    to_gpu_extras = [
        "vector_in",
        "img_in",
        "txt_in",
        "time_in",
        "guidance_in",
        "final_layer",
        "pe_embedder",
    ]
    for extra in to_gpu_extras:
        getattr(flow_model, extra).cuda(flux_device).type(flux_dtype)
````

-   Code for the main entry point:

```59:85:main.py
def main():
    args = parse_args()

    if args.config_path:
        app.state.model = load_pipeline_from_config_path(args.config_path)
    else:
        model_version = (
            ModelVersion.flux_dev
            if args.model_version == "flux-dev"
            else ModelVersion.flux_schnell
        )
        config = load_config(
            model_version,
            flux_path=args.flow_model_path,
            flux_device=args.flux_device,
            ae_path=args.autoencoder_path,
            ae_device=args.autoencoder_device,
            text_enc_path=args.text_enc_path,
            text_enc_device=args.text_enc_device,
            flow_dtype="float16",
            text_enc_dtype="bfloat16",
            ae_dtype="bfloat16",
            num_to_quant=args.num_to_quant,
        )
        app.state.model = load_pipeline_from_config(config)

    uvicorn.run(app, host=args.host, port=args.port)
```

-   Code for the API endpoint:

```22:25:api.py
@app.post("/generate")
def generate(args: GenerateArgs):
    result = app.state.model.generate(**args.model_dump())
    return StreamingResponse(result, media_type="image/jpeg")
```
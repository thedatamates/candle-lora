# candle-lora
[![MIT License](https://img.shields.io/badge/License-MIT-informational)](LICENSE)
[![Continuous integration](https://github.com/EricLBuehler/candle-lora/actions/workflows/ci.yml/badge.svg)](https://github.com/EricLBuehler/candle-lora/actions/workflows/ci.yml)

LoRA (low rank adaptation) implemented in Rust for use with [`Candle`](https://github.com/huggingface/candle/tree/main).

It is based on HuggingFace's [`peft`](https://github.com/huggingface/peft/tree/main) library. See the original paper [here](https://arxiv.org/pdf/2106.09685.pdf). 

All conversions are done as implemented in HuggingFace's official LoRA implementation.

Specifically, `candle-lora` is able to convert `Linear`, `Conv1d`, `Conv2d`, and `Embedding` into their respective LoRA counterparts. To improve inference performance, both merging and unmerging LoRA weights are also implemented.


## How to use
1) In your model structs, replace any concrete `Linear`, `Conv1d`, `Conv2d`, or `Embedding` types with `Box<dyn ...LayerLike>`. This will allow `candle-lora` to
generate new layers that can easily be swapped out without forcing you to redefine your model structs.
2) Select the layers and perform the conversion.
3) Swap out the layers.
4) Enjoy your new LoRA model!

## candle-lora-macro
Because swapping out and selecting layers is tedious and can be error-prone for medium to large models, I am developing a [crate](https://github.com/EricLBuehler/candle-lora-macro) that automates the process.

## Example
```rust
use std::{collections::HashMap, hash::Hash};

use candle_core::{DType, Device, Result, Tensor};
use candle_lora::{LinearLayerLike, Lora, LoraLinearConfig, NewLayers, SelectedLayersBuilder, LoraConfig};
use candle_nn::{init, Linear, Module, VarMap};

#[derive(PartialEq, Eq, Hash)]
enum ModelLayers {
    Layer,
}

#[derive(Debug)]
struct Model {
    layer: Box<dyn LinearLayerLike>,
}

impl Module for Model {
    fn forward(&self, input: &Tensor) -> Result<Tensor> {
        self.layer.forward(input)
    }
}

impl Model {
    fn insert_new(&mut self, new: NewLayers<ModelLayers>) {
        for (name, linear) in new.linear {
            match name {
                ModelLayers::Layer => self.layer = Box::new(linear),
            }
        }
    }
}

fn main() -> candle_core::Result<()> {
    let device = Device::Cpu;
    let dtype = DType::F32;

    //Create the model
    let map = VarMap::new();
    let layer_weight = map.get(
        (10, 10),
        "layer.weight",
        init::DEFAULT_KAIMING_NORMAL,
        dtype,
        &device,
    )?;

    let mut model = Model {
        layer: Box::new(Linear::new(layer_weight.clone(), None)),
    };

    let dummy_image = Tensor::zeros((10, 10), DType::F32, &device)?;

    //Test the model
    let digit = model.forward(&dummy_image).unwrap();
    println!("Output: {digit:?}");

    //Select layers we want to convert
    let mut linear_layers = HashMap::new();
    linear_layers.insert(ModelLayers::Layer, &*model.layer);
    let selected = SelectedLayersBuilder::new()
        .add_linear_layers(linear_layers, LoraLinearConfig::new(10, 10))
        .build();

    let loraconfig = LoraConfig::new(1, 1., None, &device, dtype);

    //Create new LoRA layers from our layers
    let new_layers = Lora::convert_model(selected, loraconfig);

    //Custom methods to implement
    model.insert_new(new_layers);

    //Test the model
    let digit = model.forward(&dummy_image).unwrap();
    println!("LoRA Output: {digit:?}");

    Ok(())
}
```
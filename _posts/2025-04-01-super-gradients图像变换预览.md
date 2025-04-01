---
layout: article
title: super-gradients图像变换预览
tags: AI cv super-gradients yolo
key: 2025-03-21-super-gradients-transform-preview
---

## 背景

在做cv训练时，经常需要对图像变换做调整，需要一个可视化的变换前后图像对比

## 实现

```python
import base64
from typing import List, Union
from io import BytesIO
from PIL import Image

import numpy as np
import torch
from torch.utils.data import DataLoader

from PIL import Image
from super_gradients.training.datasets.detection_datasets.coco_format_detection import (
    COCOFormatDetectionDataset,
)
from super_gradients.training.transforms.transforms import (
    DetectionRandomAffine,
    DetectionHSV,
    DetectionHorizontalFlip,
    DetectionPaddedRescale,
    DetectionStandardize,
    DetectionTargetsFormatTransform,
)


class COCODetectionPreviewCollateFN:
    def __init__(self):
        self.expected_item_names = ("index", "image")

    def __call__(self, data):
        # 只返回索引和变换后的图片 忽略标注
        index_batch, images_batch, _ = list(zip(*data))
        return (self._format_index(index_batch), self._format_images(images_batch))

    @staticmethod
    def _format_index(index_batch: List[int]) -> torch.Tensor:
        return torch.tensor(index_batch)

    @staticmethod
    def _format_images(
        images_batch: List[Union[torch.Tensor, np.array]],
    ) -> torch.Tensor:
        images_batch = [torch.tensor(img) for img in images_batch]
        images_batch_stack = torch.stack(images_batch, 0)
        if images_batch_stack.shape[3] == 3:
            images_batch_stack = torch.moveaxis(images_batch_stack, -1, 1).float()
        return images_batch_stack


class COCOFormatDetectionPreviewDataset(COCOFormatDetectionDataset):
    def __getitem__(self, index):
        image, target = super().__getitem__(index)
        # 同时将索引返回
        return index, image, target


def preview(dataset_dir: str, images_dir: str, annotation_file: str, params: dict):
    dataset = COCOFormatDetectionPreviewDataset(
        data_dir=dataset_dir,
        images_dir=images_dir,
        json_annotation_file=annotation_file,
        cache_annotations=False,
        input_dim=parse_tuple(params.get("input_dim")),
        ignore_empty_annotations=params.get("ignore_empty_annotations"),
        with_crowd=params.get("with_crowd"),
        transforms=generate_transforms(params.get("transforms", [])),
    )
    loader = DataLoader(dataset, collate_fn=COCODetectionPreviewCollateFN())
    indices, images = next(iter(loader))
    # 原始图片
    index = indices[0].item()
    origin_image = dataset.get_sample(index)["image"]
    # 转换后的图片
    transformed_image = images[0].numpy().transpose(1, 2, 0)
    transformed_image = np.clip(transformed_image * 255, 0, 255).astype(np.uint8)

    return {
        "before": image_to_base64(origin_image),
        "after": image_to_base64(transformed_image),
    }

def generate_transforms(transforms: list[dict]) -> list:
    result = []
    for transform in transforms:
        params = deepcopy(transform["params"])
        match transform["name"]:
            case "DetectionRandomAffine":
                format_transforms_params(
                    params,
                    ["degrees", "translate", "scales", "shear", "target_size"],
                )
                result.append(DetectionRandomAffine(**params))
            case "DetectionHSV":
                format_transforms_params(params, ["bgr_channels"])
                result.append(DetectionHSV(**params))
            case "DetectionHorizontalFlip":
                result.append(DetectionHorizontalFlip(**params))
            case "DetectionPaddedRescale":
                format_transforms_params(params, ["input_dim", "swap"])
                result.append(DetectionPaddedRescale(**params))
            case "DetectionStandardize":
                result.append(DetectionStandardize(**params))
            case "DetectionTargetsFormatTransform":
                format_transforms_params(params, ["input_dim"])
                result.append(DetectionTargetsFormatTransform(**params))
    return result


def format_transforms_params(params: dict, keys: list[str]):
    for key in keys:
        if key in params:
            params[key] = parse_tuple(params[key])


def parse_tuple(data):
    if isinstance(data, list):
        return tuple(data)
    return data

def image_to_base64(image):
    img = Image.fromarray(image)
    buf = BytesIO()
    img.save(buf, format="PNG")
    return base64.b64encode(buf.getvalue()).decode("utf-8")

def save_image(image, path):
    Image.fromarray(image).save(path)
```

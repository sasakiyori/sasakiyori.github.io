---
layout: article
title: super-gradients使用
tags: AI cv super-gradients yolo
key: 2025-03-21-super-gradients-usage
---

## 背景

[super-gradients](https://github.com/Deci-AI/super-gradients)，一个做CV相关的模型训练和微调的库

## 训练格式

支持多种格式，以下默认采用COCO格式

- <https://albumentations.ai/docs/getting_started/bounding_boxes_augmentation/#coco>

## 基础训练方式

参考 <https://github.com/Deci-AI/super-gradients/blob/master/notebooks/yolo_nas_custom_dataset_fine_tuning_with_qat.ipynb>

一些额外的注意点：

```python
train_dataset_params = {
    "data_dir": "./datasets",
    "images_dir": "images/train",
    "json_annotation_file": "train_annotations.coco.json",
    "input_dim": (640, 640),
    "ignore_empty_annotations": False,
    "with_crowd": False,
    "all_classes_list": class_names,
    "transforms": [
        DetectionRandomAffine(
            degrees=0.0,
            scales=(0.5, 1.5),
            shear=0.0,
            target_size=(640, 640),
            filter_box_candidates=False,
            border_value=128,
        ),
        DetectionHSV(prob=1.0, hgain=5, vgain=30, sgain=30),
        DetectionHorizontalFlip(prob=0.5),
        DetectionPaddedRescale(input_dim=(640, 640)),
        DetectionStandardize(max_value=255),
        DetectionTargetsFormatTransform(
            input_dim=(640, 640), output_format="LABEL_CXCYWH"
        ),
    ],
}

val_dataset_params = dict(
    data_dir="./datasets",
    images_dir="images/test",
    json_annotation_file="test_annotations.coco.json",
    input_dim=(640, 640),
    ignore_empty_annotations=False,
    with_crowd=False,
    all_classes_list=class_names,
    transforms=[
        # 用来做验证评估的transforms比train的少，这是因为有的transform方法没有实现get_equivalent_preprocessing方法
        # 一旦加了未实现get_equivalent_preprocessing的transofrm方法，会导致最终生成的model没有class_names和image_processor信息
        # 这会导致使用model进行predict时，你还得手动添加图片处理方法 (尽可能与训练时保持一致)
        DetectionPaddedRescale(input_dim=(640, 640), max_targets=300),
        DetectionStandardize(max_value=255),
        DetectionTargetsFormatTransform(
            input_dim=(640, 640), output_format="LABEL_CXCYWH"
        ),
    ],
)
```

```python
train_args = {
    "warmup_initial_lr": 1e-5,
    "initial_lr": 5e-4,
    "lr_mode": "cosine",
    "cosine_final_lr_ratio": 0.5,
    "optimizer": "AdamW",
    "zero_weight_decay_on_bias_and_bn": True,
    "lr_warmup_epochs": 1,
    "warmup_mode": "LinearEpochLRWarmup",
    "optimizer_params": {"weight_decay": 0.0001},
    "ema": False,
    "average_best_models": True,
    "ema_params": {"beta": 25, "decay_type": "exp"},
    "max_epochs": 10,
    "mixed_precision": True,
    # 这里的loss function需要与自己要训练的模型对应
    # 具体对应逻辑在 https://github.com/Deci-AI/super-gradients/blob/master/documentation/source/ObjectDetection.md
    "loss": "PPYoloELoss",
    "criterion_params": {
        "num_classes": num_classes,
    },
    "valid_metrics_list": [
        # 这里的watch类型需要与自己要训练的模型对应
        # 例如yolo_nas是detection类型的，yolo_nas_pose是pose estimation类型的
        DetectionMetrics_050(
            score_thres=0.1,
            top_k_predictions=300,
            num_cls=num_classes,
            normalize_targets=True,
            include_classwise_ap=True,
            class_names=class_names,
            # 这里的callback需要与loss function对应
            # 具体对应逻辑在 https://github.com/Deci-AI/super-gradients/blob/master/documentation/source/ObjectDetection.md
            post_prediction_callback=PPYoloEPostPredictionCallback(
                score_threshold=0.01,
                nms_top_k=1000,
                max_predictions=300,
                nms_threshold=0.7,
            ),
        ),
    ],
    # 这里的watch的枚举是由loss function和valid_metrics_list共同决定的
    # 对于有实现component_names方法的loss function，枚举列表类似于：
    # ["PPYoloELoss" + "/" + c for c in PPYoloELoss(**kwargs).component_names]
    # 对于valid_metrics_list，枚举列表类似于：
    # get_metrics_titles(MetricCollection(valid_metrics_list))
    "metric_to_watch": "mAP@0.50",
}
```

```python
PRETRAINED_MODEL_PATH = "/path/to/model"

pretrained_state_dict = torch.load(PRETRAINED_MODEL_PATH, weights_only=True)
net = pretrained_state_dict.get("net", {})
head = net.get("heads.head1.cls_pred.weight")
shape = head.shape
pretrained_num_classes = shape[0]

model = models.get(
    model_name=Models.YOLO_NAS_S,
    arch_params=None,
    num_classes=num_classes,
    checkpoint_path=PRETRAINED_MODEL_PATH,
    # 这边不能直接用了 因为官方似乎已经停止维护 而这边传入权重名字所对应的默认下载路径已经访问不通
    # 可以考虑到 https://github.com/Deci-AI/super-gradients/blob/master/src/super_gradients/training/pretrained_models.py 下载
    # 然后使用checkpoint_path传入
    pretrained_weights=None,
    # 如果加载了预训练模型 则需要判断这个预训练模型是以多少个分类进行训练的
    # 这边需要传入它的层数以防两个num_classes不一致 否则会报错
    checkpoint_num_classes=pretrained_num_classes,
)
```

## 自定义数据集处理器

```python
import os
from typing import Any, Dict, Tuple, List, Union

import cv2
from datasets import load_dataset
import numpy as np
from super_gradients.common.object_names import ConcatenatedTensorFormats
from super_gradients.training.datasets.detection_datasets.detection_dataset import (
    DetectionDataset,
)
from super_gradients.training.transforms.transforms import DetectionTransform
from super_gradients.training.utils.detection_utils import (
    change_bbox_bounds_for_image_size,
)

class SuperGradientsDetectionDataset(DetectionDataset):
    def __init__(
        self,
        data_dir: str,
        tmp_image_dir: str,
        split: str = "train",
        origin_bbox_format: str = "XYXY",
        max_example_cnt: int = None,
        input_dim: Union[int, Tuple[int, int], None] = None,
        transforms: List[DetectionTransform] = [],
        **kwargs,
    ):
        self.data_dir = data_dir
        self.tmp_image_dir = tmp_image_dir
        self.origin_bbox_format = origin_bbox_format
        self.input_dim = input_dim
        self.annotations = []

        os.makedirs(os.path.join(data_dir, tmp_image_dir), exist_ok=True)

        self.datasets = load_dataset(data_dir, split=split)
        # 这里根据实际的数据集确定获取方式
        self.class_names = self.datasets.features["objects"].feature["category"].names

        # 在最后才初始化基类
        super().__init__(
            data_dir=data_dir,
            input_dim=input_dim,
            # 固定使用XYXY格式 数据集在加载过程中手动转换
            original_target_format=ConcatenatedTensorFormats.XYXY_LABEL,
            max_num_samples=max_example_cnt,
            transforms=transforms,
            all_classes_list=self.class_names,
            **kwargs,
        )

    # 必须实现的虚函数
    def _setup_data_source(self) -> int:
        for sample in self.datasets:
            # 这里根据实际的数据集确定获取方式
            image_id = sample["image_id"]
            image = np.array(sample["image"])
            image_width = sample["width"]
            image_height = sample["height"]
            bboxes = sample["objects"]["bbox"]
            labels = sample["objects"]["category"]
            crowds = [0 for _ in labels]

            filename = f"{image_id}.jpg"
            image_path = os.path.join(self.data_dir, self.tmp_image_dir, filename)
            # RGB to BGR
            # 如果不使用opencv则不需要转换
            cv2.imwrite(image_path, image[..., ::-1])

            match self.origin_bbox_format:
                case "XYWH":
                    bboxes = [self.xywh_to_xyxy(bbox) for bbox in bboxes]
                case "CXCYWH":
                    bboxes = [self.cxcywh_to_xyxy(bbox) for bbox in bboxes]

            self.annotations.append(
                {
                    "image_id": image_id,
                    "img_path": image_path,
                    "image_width": image_width,
                    "image_height": image_height,
                    "bbox": np.asarray(bboxes, dtype=np.float32).reshape(-1, 4),
                    "crowd": np.asarray(crowds, dtype=bool).reshape(-1),
                    "labels": np.asarray(labels, dtype=int).reshape(-1),
                }
            )

        del self.datasets
        return len(self.annotations)

    # 必须实现的虚函数
    def _load_annotation(self, sample_id: int) -> Dict[str, Union[np.ndarray, Any]]:
        annotation = self.annotations[sample_id]

        width = annotation["image_width"]
        height = annotation["image_height"]
        boxes_xyxy = change_bbox_bounds_for_image_size(
            annotation["bbox"], img_shape=(height, width), inplace=False
        )
        crowd = annotation["crowd"].copy()
        labels = annotation["labels"].copy()

        # 去掉不符合bbox格式要求的数据
        mask = np.logical_and(
            boxes_xyxy[:, 2] >= boxes_xyxy[:, 0], boxes_xyxy[:, 3] >= boxes_xyxy[:, 1]
        )
        boxes_xyxy = boxes_xyxy[mask]
        crowd = crowd[mask]
        labels = labels[mask]

        initial_img_shape = (height, width)
        if self.input_dim is not None:
            scale_factor = min(self.input_dim[0] / height, self.input_dim[1] / width)
            resized_img_shape = (int(height * scale_factor), int(width * scale_factor))
        else:
            resized_img_shape = initial_img_shape
            scale_factor = 1

        # 在这里做最后的拼接 将分类拼接上来
        # 由XYXY变成XYXY_LABEL 满足我们初始化时设定的original_target_format
        targets = np.concatenate(
            [boxes_xyxy[~crowd] * scale_factor, labels[~crowd, None]], axis=1
        ).astype(np.float32)
        crowd_targets = np.concatenate(
            [boxes_xyxy[crowd] * scale_factor, labels[crowd, None]], axis=1
        ).astype(np.float32)

        ann = {
            "target": targets,
            "crowd_target": crowd_targets,
            "initial_img_shape": initial_img_shape,
            "resized_img_shape": resized_img_shape,
            "img_path": annotation["img_path"],
        }
        return ann

    # 虚函数，非必要，但是最好实现一下
    @property
    def _all_classes(self):
        return self.class_names
    
    def xywh_to_xyxy(self, bbox: list):
        x1 = bbox[0]
        y1 = bbox[1]
        x2 = bbox[0] + bbox[2]
        y2 = bbox[1] + bbox[3]
        return [x1, y1, x2, y2]


    def cxcywh_to_xyxy(self, bbox: list):
        x1 = bbox[0] - bbox[2] / 2
        y1 = bbox[1] - bbox[3] / 2
        x2 = bbox[0] + bbox[2] / 2
        y2 = bbox[1] + bbox[3] / 2
        return [x1, y1, x2, y2]
```

## 自定义训练数据采集

```python
# 注册完之后，在training_params里设置 sg_logger=my_sg_reporter 即可
@register_sg_logger("my_sg_reporter")
class MyReporter(BaseSGLogger):
    def __init__(
        self,
        project_name: str,
        experiment_name: str,
        storage_location: str,
        resumed: bool,
        training_params: TrainingParams,
        checkpoints_dir_path: str,
        tb_files_user_prompt: bool = False,
        launch_tensorboard: bool = False,
        tensorboard_port: int = None,
        save_checkpoints_remote: bool = True,
        save_tensorboard_remote: bool = True,
        save_logs_remote: bool = True,
        monitor_system: bool = True,
    ):
        super().__init__(
            project_name=project_name,
            experiment_name=experiment_name,
            storage_location=storage_location,
            resumed=resumed,
            training_params=training_params,
            checkpoints_dir_path=checkpoints_dir_path,
            tb_files_user_prompt=tb_files_user_prompt,
            launch_tensorboard=launch_tensorboard,
            tensorboard_port=tensorboard_port,
            save_checkpoints_remote=save_checkpoints_remote,
            save_tensorboard_remote=save_tensorboard_remote,
            save_logs_remote=save_logs_remote,
            monitor_system=monitor_system,
        )
    
    @multi_process_safe
    def add_scalar(
        self, tag: str, scalar_value: float, global_step: Union[int, TimeUnit] = None
    ):
        super().add_scalar(tag, scalar_value, global_step)
        # do something
    
    @multi_process_safe
    def add_scalars(self, tag_scalar_dict: dict, global_step: int = None):
        super().add_scalars(tag_scalar_dict, global_step)
        # do something
    
    @multi_process_safe
    def add_text(self, tag: str, text_string: str, global_step: int = None):
        super().add_text(tag, text_string, global_step)
        # do something
    
    @multi_process_safe
    def close(self):
        super().close()
        # do something
```

## 自定义训练流程控制器

```python
# 注册完之后，在training_params里设置 phase_callbacks=[MyCallback()] 即可
@register_callback()
class MyCallback(Callback):
    def on_training_start(self, context: PhaseContext) -> None:
        # do something
    
    def on_train_loader_start(self, context: PhaseContext) -> None:
        # do something
    
    # ...

    def on_train_loader_end(self, context: PhaseContext) -> None:
        # 可以异步停止训练
        # 但是并不是每个地方设置停止都会生效 目前应该是一个epoch后完成会停止
        # https://github.com/Deci-AI/super-gradients/issues/1151
        context.update_context(stop_training=True)
    
    # ...
```

## 图片预测

```python
# 模型加载用训练时的加载方式即可
# 模型训练正确的话 是不需要额外设置image_processor的
prediction = model.predict(IMAGE_PATH, fuse_model=False)

prediction.show()

prediction.save(SAVE_PATH)
```

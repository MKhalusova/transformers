<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.

-->

# Visual Question Answering

[[open-in-colab]]

Visual Question Answering (VQA) is the task of answering open-ended questions based on an image. 
The input to models supporting this task is typically a combination of an image and a question, and the output is an 
answer expressed in natural language.

Some noteworthy use case examples for VQA include:
* Accessibility applications for visually impaired individuals.
* Education: posing questions about visual materials presented in lectures or textbooks. VQA can also be utilized in interactive museum exhibits or historical sites.
* Customer service and e-commerce: VQA can enhance user experience by letting users ask questions about products. 
* Image retrieval: VQA models can be used to retrieve images with specific characteristics. For example, the user can ask "Is there a dog?" to find all images with dogs from a set of images.

In this guide you'll learn how to:

- Fine-tune a VQA model, specifically [ViLT](../model_doc/vilt), on the [`Graphcore/vqa` dataset](https://huggingface.co/datasets/Graphcore/vqa).
- Use your fine-tuned model for inference.

ViLT model incorporates text embeddings into a Vision Transformer (ViT), allowing it to have a minimal design for 
Vision-and-Language Pre-training (VLP). This model can be used for several downstream tasks. For the VQA task, a classifier 
head is placed on top (a linear layer on top of the final hidden state of the [CLS] token) and randomly initialized. 
Visual Question Answering is thus treated as a classification problem.

Before you begin, make sure you have all the necessary libraries installed. 

```bash
pip install -q transformers datasets
```

We encourage you to share your model with the community. Log in to your Hugging Face account to upload it to the 🤗 Hub.
When prompted, enter your token to log in:

```py
>>> from huggingface_hub import notebook_login

>>> notebook_login()
```

Let's define the model checkpoint as a global variable.

```py
>>> model_checkpoint = "dandelin/vilt-b32-mlm"
```

## Load the data

For illustration purposes, in this guide we use a very small sample of the annotated visual question answering `Graphcore/vqa` dataset. 
You can find the full dataset on [🤗 Hub](https://huggingface.co/datasets/Graphcore/vqa).

Let's load the first 200 examples from the validation split and explore the dataset's features:  

```python
>>> from datasets import load_dataset

>>> dataset = load_dataset("Graphcore/vqa", split="validation[:200]")
>>> dataset
Dataset({
    features: ['question', 'question_type', 'question_id', 'image_id', 'answer_type', 'label'],
    num_rows: 200
})
```

Let's take a look at an example to understand the dataset's features:

```py
>>> dataset[0]
{'question': 'Where is he looking?',
 'question_type': 'none of the above',
 'question_id': 262148000,
 'image_id': '/root/.cache/huggingface/datasets/downloads/extracted/ca733e0e000fb2d7a09fbcc94dbfe7b5a30750681d0e965f8e0a23b1c2f98c75/val2014/COCO_val2014_000000262148.jpg',
 'answer_type': 'other',
 'label': {'ids': ['at table', 'down', 'skateboard', 'table'],
  'weights': [0.30000001192092896,
   1.0,
   0.30000001192092896,
   0.30000001192092896]}}
```

The features relevant to the task include: 
* `question`: the question to be answered from the image
* `image_id`: the path to the image the question refers to
* `label`: the annotations

We can remove the rest of the features as they won't be necessary: 

```py 
>>> dataset = dataset.remove_columns(['question_type', 'question_id', 'answer_type'])
```

As you can see, the `label` feature contains several answers to the same question (called `ids` here) collected by different human annotators. 
This is because the answer to a question can be subjective. In this case, the question is "where is he looking?". Some people 
annotated this with "down", others with "at table", another one with "skateboard", etc. 

Take a look at the image and consider which answer would you give:

```python
>>> from PIL import Image

>>> image = Image.open(dataset[0]['image_id'])
>>> image
```

[//]: # (TODO: add the image)

<div class="flex justify-center">
     <img src="" alt="VQA Image Example"/>
</div>

Due to the questions and answers ambiguity, datasets like this are treated as a multi-label classification problem (as 
multiple answers are possibly valid). Moreover, rather than just creating a one-hot encoded vector, one creates a 
soft encoding, based on the number of times a certain answer appeared in the annotations.

For instance, in the example above, because the answer "down" is selected way more often than other answers, it has a 
score (called `weight` in the dataset) of 1.0, and the rest of the answers have scores < 1.0. 

To later instantiate the model with an appropriate classification head, let's create two dictionaries: one that maps 
the label name to an integer and vice versa:

```py
>>> import itertools

>>> labels = [item['ids'] for item in dataset['label']]
>>> flattened_labels = list(itertools.chain(*labels))
>>> unique_labels = list(set(flattened_labels))

>>> label2id = {label: idx for idx, label in enumerate(unique_labels)}
>>> id2label = {idx: label for label, idx in label2id.items()} 
```

Now that we have the mappings, we can replace the string answers with their ids, and flatten the dataset for a more convenient further preprocessing. 

```python
>>> def replace_ids(inputs):
...   inputs["label"]["ids"] = [label2id[x] for x in inputs["label"]["ids"]]
...   return inputs


>>> dataset = dataset.map(replace_ids)
>>> flat_dataset = dataset.flatten()
>>> flat_dataset.features
{'question': Value(dtype='string', id=None),
 'image_id': Value(dtype='string', id=None),
 'label.ids': Sequence(feature=Value(dtype='int64', id=None), length=-1, id=None),
 'label.weights': Sequence(feature=Value(dtype='float64', id=None), length=-1, id=None)}
```

## Preprocessing data

The next step is to load a ViLT processor to prepare the image and text data for the model. 
[`ViLTProcessor`] wraps a BERT tokenizer and ViLT image processor into a convenient single processor:

```py 
>>> from transformers import ViltProcessor

>>> processor = ViltProcessor.from_pretrained(model_checkpoint)
```

To preprocess the data we need to encode the images and questions using the [`ViLTProcessor`]. The processor will use 
the [`BertTokenizerFast`] to tokenize the text and create `input_ids`, `attention_mask` and `token_type_ids` for the text data. 
As for images, the processor will leverage [`ViltFeatureExtractor`] to resize and normalize the image, and create `pixel_values` and `pixel_mask`.

All these preprocessing steps are done under the hood, we only need to call the `processor`. However, we still need to 
prepare the target labels. Labels are represented as a soft encoded vector where of shape (num_labels,) where the each 
element represents a potential answer (label): for valid answers the element contains their score (weight), and the rest are zeroes.

The following function applies the `processor` to the images and questions and formats the labels as described above:

```py
>>> import torch

>>> def preprocess_data(examples):
...     image_paths = examples['image_id']
...     images = [Image.open(image_path) for image_path in image_paths]
...     texts = examples['question']    

...     encoding = processor(images, texts, padding="max_length", truncation=True, return_tensors="pt")

...     for k,v in encoding.items():
...           encoding[k] = v.squeeze()
    
...     targets = []

...     for labels, scores in zip(examples['label.ids'], examples['label.weights']):
...       target = torch.zeros(len(id2label))

...       for label, score in zip(labels, scores):
...         target[label] = score
      
...       targets.append(target)

...     encoding["labels"] = targets
    
...     return encoding
```

To apply the preprocessing function over the entire dataset, use 🤗 Datasets map function. You can speed up map by 
setting `batched=True` to process multiple elements of the dataset at once. At this point, feel free to remove the columns you don't need.

```py
>>> processed_dataset = flat_dataset.map(preprocess_data, batched=True, remove_columns=['question','question_type', 'question_id', 'image_id', 'answer_type', 'label.ids', 'label.weights'])
>>> processed_dataset
Dataset({
    features: ['input_ids', 'token_type_ids', 'attention_mask', 'pixel_values', 'pixel_mask', 'labels'],
    num_rows: 200
})
```

As a final preprocessing step, create a batch of examples using [`DefaultDataCollator`]:

```py
>>> from transformers import DefaultDataCollator

>>> data_collator = DefaultDataCollator()
```

## Train the model

You’re ready to start training your model now! Load ViLT with `ViltForQuestionAnswering`. Specify the number of labels 
along with the label mappings:

```py
>>> from transformers import ViltForQuestionAnswering

>>> model = ViltForQuestionAnswering.from_pretrained(model_checkpoint, num_labels=len(id2label), id2label=id2label, label2id=label2id)
```

At this point, only three steps remain:

1. Define your training hyperparameters in [`TrainingArguments`]:

```py
>>> from transformers import TrainingArguments

>>> repo_id = "MariaK/vilt_finetuned_200"

>>> training_args = TrainingArguments(
...     output_dir=repo_id,
...     per_device_train_batch_size=4,
...     num_train_epochs=20,
...     save_steps=200,
...     logging_steps=50,
...     learning_rate=5e-5,
...     save_total_limit=2,
...     remove_unused_columns=False,
...     push_to_hub=True,
... )
```

2. Pass the training arguments to [`Trainer`] along with the model, dataset, processpe, and data collator.

```py
>>> from transformers import Trainer

>>> trainer = Trainer(
...     model=model,
...     args=training_args,
...     data_collator=data_collator,
...     train_dataset=processed_dataset,
...     tokenizer=processor,
... )
```

3.Call `train` to finetune your model.

```py
>>> trainer.train() 
```

Once training is completed, share your model to the Hub with the `push_to_hub` method to share your final model on the 🤗 Hub:

```py
>>> trainer.push_to_hub()
```

## Inference

Now that you have fine-tuned a ViLT model, and uploaded it to the 🤗 Hub, you can use it for inference. The simplest
way to try out your fine-tuned model for inference is to use it in a [`Pipeline`].

```py
>>> from transformers import pipeline

>>> pipe = pipeline("visual-question-answering", model="MariaK/vilt_finetuned_200")
```

The model in this guide has only been trained on 200 examples, so don't expect a lot of it. Let's see if it at least 
learned something from the data and take the first example from the dataset to illustrate inference:

```py
>>> example = dataset[0]
>>> image = Image.open(example['image_id'])
>>> question = example['question']
>>> print(question)
>>> pipe(image, question, top_k=1)
"Where is he looking?"
[{'score': 0.5498199462890625, 'answer': 'down'}]
```

Even though not very confident, the model indeed has learned something. With more examples and longer training, you'll get far better results!

You can also manually replicate the results of the pipeline if you'd like:
1. Take an image and a question, prepare them for the model using the processor from your model.
2. Forward the result or preprocessing through the model.
3. From the logits, get the most likely answer's id, and find the actual answer in the `id2label`.

```py
>>> processor = ViltProcessor.from_pretrained("MariaK/vilt_finetuned_200")

>>> image = Image.open(example['image_id'])
>>> question = example['question']

>>> # prepare inputs
>>> inputs = processor(image, question, return_tensors="pt")

>>> model = ViltForQuestionAnswering.from_pretrained("MariaK/vilt_finetuned_200")

>>> # forward pass
>>> with torch.no_grad():
...     outputs = model(**inputs)

>>> logits = outputs.logits
>>> idx = logits.argmax(-1).item()
>>> print("Predicted answer:", model.config.id2label[idx])
Predicted answer: down
```

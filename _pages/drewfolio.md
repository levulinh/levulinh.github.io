---
layout: page
permalink: /drew-folio/
title: portfolio
description: Look what I have done!
nav: true
nav_order: 3
---


## Project 0: Dynamic Graph Convolutional Neural Network for Image Classification on superpixel images

This is my first publication during my master's degree (Hence the number 0). Despite of a very rushing deadline, I managed to pull it off, and I am proud of it. The project was about using a dynamic graph convolutional neural network to classify superpixel images. The model was trained on multiple datasets, including CIFAR, Fashion-MNIST, superpixel-MNIST. And the superpixel images were generated using the [SLIC](https://scikit-image.org/docs/dev/auto_examples/segmentation/plot_segmentations.html) algorithm.

In this project, I proposed a new method to generate graphs on superpixel images dynamically every iteration, mitigating
the problem of graph saturation, and increasing the receptive field of the model. The method out-performed the SOTA methods on the same setting on multiple datasets.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/dynamic_gnn.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

Perfomance comparison:

- on MNIST-75

|  | MNIST-75 |
|---------|-------------|
| MoNET | 91.11% |
| SplineCNN | 95.22% |
| GeoGCN | 95.95% |
| RAG-GAT | 96.19% |
| **Ours** | **99.04%** |
|---------|-------------|


- on Fashion-MNIST

|  | Fashion-MNIST |
|---------|-------------|
| RAG-GAT | 83.07% |
| **Ours** | **90.02%** |
|---------|-------------|

### Pulication:
[Dynamic Graph Neural Network for Super-Pixel Image Classification](https://ieeexplore.ieee.org/abstract/document/9621101)

### Technologies Used:
- PyTorch
- PyTorch Geometric
- SLIC

### Repository:
A repository of a similar project that I did for remote sensing data, the technique is the same, but the data are
real-word data.
{% include repository/repo.html repository="levulinh/remote-sensing-dynamic-spixel" %}

---

## Project 1: Multi-task Distillation learning for Image

Context: So, let's rewind a bit and talk about the sleep monitoring gig at Asleep. We decided to spice things up by throwing Vision Transformer (ViT) models into the mix. Forget the typical Recurrent Neural Network (RNN) routine – we took a different route, predicting sleep stages, catching apnea events, and eavesdropping on snores using ViT models. The twist? We didn't just toss raw data at our AI; we served it up with transformed Mel-spectrograms of breathing sounds. Why ViT? Picture a sleep detective that hones in on the nitty-gritty details in those spectrograms, making our predictions spot-on. No shade to the RNN crew, but ViT not only steals the spotlight but does it in real-time fashion. It's been a thrilling journey, and we're rewriting the playbook on sleep monitoring – one ViT model at a time!

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/sound_to_mel.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/two_stage.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    The overall architecture of our sound-based models. For more information, please refer to the ICLR <a href="https://openreview.net/pdf?id=mIRztWMsVJ">Workshop paper</a>.
</div>

### Then what's the problem?
- **Task Segregation Woes:**
Although our sleep monitoring tasks appeared closely related, they were operating in separate spheres, leading to a missed opportunity for potential performance and speed enhancement.

- **Multi-Task Model Dilemma:**
Building a multi-task model seemed like the logical next step, but the real challenge lay in the intricacies of training. Integrating tasks might sound easy, but figuring out the most effective training strategy proved to be a different story altogether.

- **The Naive Approach Pitfall:**
In attempting to streamline our efforts, we naively opted for training the model with supervised learning. However, the outcome was far from ideal – the performance of at least one task took a nosedive, a twist we hadn't quite anticipated. Stick around as I unravel the complexities and showcase the unexpected hurdles in achieving seamless integration.

### The Solution: Multi-Task Distillation Learning

- **The Big Picture:**
You can't expect both (or more) tasks to keep the same performance by only training them supervisedly, and simultaneously. The solution is to train the model with a combination of supervised and distillation learning. The supervised learning part is to keep the performance of each task, and the distillation learning part is to make the model learn from other tasks, which is the key to the whole thing.

The whole idea is inspired by the paper [Knowledge Distillation for Multi-task Learning](https://arxiv.org/abs/2007.06889).

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/multi_head.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Overall architecture of the multi-task model.
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/multi_head_train.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Oversimplified training process of the multi-task model.
</div>

But, there is one more thing. The method proposed by the paper did not take into account the difficulty of adapting to the new tasks from the same root. So, I made a modification to the method, and it worked like a charm.
I moved the distillation loss to the beginning of the split heads, so that the model can adapt the changes in the new tasks faster.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/multi_head_train_2.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

Due to the limitation of what I can expose, the only thing that I can provide is the performance of the model on the test set of all tasks surpasses the performance of the model trained with supervised learning only, and separately.

### Publications:
- [Real-Time Detection of Sleep Apnea Based on Breathing Sounds and Prediction Reinforcement Using Home Noises: Algorithm Development and Validation](https://www.jmir.org/2023/1/e44818/)
- [In-Home Smartphone-Based Prediction of Obstructive Sleep Apnea in Conjunction With Level 2 Home Polysomnography](https://jamanetwork.com/journals/jamaotolaryngology/article-abstract/2812069)
- [EVALUATION OF A SOUND-BASED DEEP LEARNING MODEL FOR HOME-BASED OBSTRUCTIVE SLEEP APNEA DETECTION USING LEVEL 2 HOME PSG DATA](https://journal.chestnet.org/article/S0012-3692(23)05087-0/fulltext)

### Technologies Used:

- PyTorch

---

## Project 2: Multiple-choice question generation by finetuning T5 model

This is a part of an assignment that I did recently. In this project, I have tried to generate Multiple-choice questions (MCQs) from a given passage. Two solutions were provided:
- Use OpenAI's GPT-3.5 model to generate the questions. This method demonstrated how we can do prompt engineering to make the model generate the questions in the format that we want. In this case, I asked the assistant to return a JSON object with the question and the choices.
- Finetune T5 model to generate the question from an answer and the context paragraph. Train another T5 model to generate the distractors for the questions. The models were trained with SQuAD v2.0 dataset for question generation, and RACE/MCTest dataset for distractor generation.

### Screenshots:
Sample paragraph:
> Bun Bo Hue is a flavorful and aromatic Vietnamese noodle soup that originates from the city of Hue in central Vietnam. Renowned for its bold and spicy profile, this soup features a robust broth made with a combination of beef and pork bones, lemongrass, shrimp paste, and a medley of aromatic spices. The dish is typically served with round rice noodles, sliced beef, pork, and sometimes cubes of congealed pig's blood. What sets Bun Bo Hue apart is its complex and rich flavor profile, characterized by the harmonious blend of lemongrass, chili, and fermented shrimp paste. Topped with fresh herbs, lime wedges, and crunchy bean sprouts, this dish offers a delightful interplay of textures and tastes, making it a beloved and distinctive part of Vietnamese cuisine. Whether enjoyed in the bustling streets of Vietnam or in Vietnamese restaurants around the world, Bun Bo Hue remains a culinary delight for those seeking a hearty and spicy noodle soup experience.

<div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/mcq_gen_1.png" title="MCQ generated by GPT3.5" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/mcq_gen_0.png" title="MCQ generated by T5 finetuned" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

### Technologies Used:
- Huggingface's Transformers
- Huggingface's Datasets
- OpenAI's GPT-3.5

### Repository:

The detail implementation can be found in the repository.
{% include repository/repo.html repository="levulinh/MCQ-generation" %}

---

## Project 3: Dog Breed (image) and News (text) Classification

This is the client for my two projects that I did in a course of my early year in master's degree. The project contains two small fun apps:
- News classification: This app is used to classify news into 4 categories: business, entertainment, politics, sport and tech.
- Dog breed classification: This app is used to classify dog breeds into 120 breeds.

The models were rather simple, but for a first year master's student, it was a good start. The models were trained using PyTorch and deployed using Flask. The client was built using ReactJS.

After years, I can definitely see a lot of things that I could have done better. But it was a good start, and I am proud of it.

### Model Architecture:
- News classification: An SetimentRNN model with 3 layers of LSTM. Using GloVe embedding, and `simple-english` tokenizer. Trained with BBC News dataset.
- Dog breed classification: A simple CNN model with 4 convolutional layers and 2 fully connected layers. Trained on a dataset of 120 dog breeds.

Both datasets were downloaded from Kaggle. And the models were trained on Google Colab (I know, I was a poor student back then 😁).

### Screenshots:

Dog breed classification
<div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/dog_breed.png" title="dog breed clf" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/dog_breed_2.png" title="example image" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>


News classification
<div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/news_clf.png" title="dog breed clf" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/news_clf_2.png" title="example image" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

### Tools Used:

- PyTorch
- Flask
- ReactJS

### Link to Project:
The client is deployed on Render, and can be accessed here:
- [Project Demo](https://what-the-dog.onrender.com)
{% include repository/repo.html repository="levulinh/SOC-Term-project-client" %}
But the server is not deployed, due to the limit of free tier on Render. If you want to see the server, you can clone the repo and run it locally.
{% include repository/repo.html repository="levulinh/SOC-Term-project-server" %}



---

## Project 4: The Flutter Vocab learning App (WIP)
This is a WIP project that I have been working on for a while. It is a simple app that helps you learn new vocabularies in a foreign language in a fun way. I am building it with Flutter, and it so far has been an exciting journey. I am still working on it, and I hope to finish it soon.

### Screenshots:
<div class="row justify-content-sm-center">
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/voca_start.png" title="example image" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/voca_home.png" title="example image" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

<div class="row justify-content-sm-center">
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/voca_remember.png" title="example image" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/voca_forget.png" title="example image" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/voca_detail.png" title="example image" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Screenshots of the application (WIP).
</div>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/openai_finetune.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">
    Model finetuning with OpenAI's GPT-3.5.
</div>

### Technologies Used:
- Flutter
- Dart
- OpenAI API

---

## Project 5: King's beat - the game 👾

Did I say I am so so so into games? 🎮 Playing them, making them, even this page's favicon is a game icon. This is a project that I tried a few months ago, just to try out the Godot game engine. It was a fun experience, and I learned a lot about game development.

### Free assets used
- [Kings and pigs](https://pixelfrog-assets.itch.io/kings-and-pigs)

### What did I build?
- A simple map from tilesets
- I drew the little score bar with Procreate
- Game logic that resembles the original game `Audition` (the legendary game that I played a lot when I was a kid)
- Scoring system

### Screenshots:
<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/portfolio/king_beat.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

<iframe width="100%" height="auto" src="https://www.youtube.com/embed/DeB7pYBUbDo" title="King beat demo video" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

And more on the github repo.

### Technologies Used:

- Godot 4.0
- GDScript

### Link to Project:

{% include repository/repo.html repository="levulinh/king-beat-game" %}
---

## Contact Me

- Email: levulinhkr@gmail.com
- LinkedIn: [Andrew Le](https://www.linkedin.com/in/andrew-le-d28m08y19/)
- GitHub: [Le Vu Linh](https://github.com/levulinh)

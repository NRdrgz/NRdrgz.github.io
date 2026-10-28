---
layout: post
title: "Monkey see, monkey do: teaching robots by demonstration with Imitation Learning"
date: 2026-10-28
thumbnail: /assets/images/posts/imitation-learning/thumbnail.jpg
hero_image: /assets/images/posts/imitation-learning/hero.jpg
excerpt: "No reward functions, no simulator. Just show the robot what to do. A practical and intuitive tour of imitation learning, the ACT model, and the road to VLAs like Pi0.5."
---

It's been a while since my last post. I was busy teaching robots to do stuff, so let's wrap our minds around it.

<video class="img-large" autoplay loop muted playsinline>
  <source src="/assets/images/posts/imitation-learning/box_opening.mp4" type="video/mp4">
</video>

<div class="story-bar">
That's my Seeed Studio robotic arm opening the door of an electrical cabinet. It never learnt in simulation. I just grabbed the arm, showed it how to do the task a few dozen times, hit train, and let it figure out the rest. <br>
<br>
After spending some time on the reinforcement learning side with my {% include link_prominent.html url="https://nrdrgz.github.io/2025/11/06/double-pendulum-isaaclab/" text="inverted double pendulum" %}, this feels almost like cheating.
</div>

In my {% include link_prominent.html url="https://nrdrgz.github.io/2025/12/02/double-pendulum-rl-deep-dive/" text="last article" %}, we taught a robot to balance a pendulum using reinforcement learning. We never told it *what* to do. We only told it *how well* it did, through a reward signal, and let it discover a strategy on its own across millions of simulation steps.

This time, we're doing the exact opposite. We're going to *show* the robot what to do.

Welcome to **imitation learning**.

## From rewards to demonstrations

Let's put them next to each other before going further.

With **reinforcement learning**, you don't need any data. But there's a catch (actually a few):
- You have to hand-craft a reward function. (Remember how much time we spent tweaking those reward weights for the pendulum?)
- You usually need a simulator, because the agent learns by trial and error, and you can't crash a real robot a million times.
- And then you let it explore for millions of steps until it stumbles onto something good.

With **imitation learning**, you flip the whole thing around. The robot learns by copying *you*. The price you pay is that now you need real-life demonstrations.

<div class="callout">
Reinforcement learning: "Here's the world. Try things. I'll tell you if you're doing well or badly." <br>
Imitation learning: "Here's how I did it. Now you do it."
</div>

That sounds much simpler, and in many ways it is. It's a big part of why robot manipulation has exploded recently. But imitation learning has its own dirty little secret, one that bit me hard at a hackathon last December. Hold that thought, we'll get to it.

## What is imitation learning, really?

At its core, the simplest form of imitation learning is just **supervised learning** wearing a robotics costume.

Think back to how we framed things last time. Supervised learning is "here's the input, here's the correct label, now learn to produce the label." Imitation learning, in its most basic form (called **behavior cloning**), is exactly that:

- **Input**: what the robot sees and feels right now. The camera image(s) and the position of each joint, sometimes more, like motor currents or force sensors.
- **Label**: the action I took at that exact moment. Where I moved each joint.

You record a bunch of `(observation, action)` pairs while doing the task, and you train a neural network to map one to the other. That's it. The robot is literally cloning your behavior, one moment at a time.

![Behavior cloning as supervised learning](/assets/images/posts/imitation-learning/observation_actions.svg){: .img-large}

There are fancier flavors of imitation learning out there (things like DAgger, or inverse reinforcement learning where you try to *recover* the reward function from demonstrations). But the models everyone is excited about right now, including the two we'll dig into, are built on good old behavior cloning. So that's what we'll focus on.

The interesting part isn't the idea, which is almost embarrassingly simple. It's everything that goes wrong when you actually try it, and the clever tricks people invented to fix it.

## Collecting demonstrations: teleoperation

Before any of that, you need data. And in robotics, the data *is* the model. There's no giant dataset of "opening electrical cabinets" sitting on the internet. You have to make it yourself.

The way you do this is **teleoperation**. You physically move the robot through the task while recording everything.

The classic setup is a **leader and a follower**. You have two arms: a leader arm that you move by hand, and a follower arm that mirrors its movements. As you guide the leader through the task, the follower copies it, and the whole time you're recording the joint positions and the camera feed. Each successful run is one "episode."

I used two different arms across my experiments:
- The **SO-101**, which I'd worked with before, has **5 degrees of freedom**.
- The **Seeed Studio arm** has **6 degrees of freedom**.

That extra degree of freedom matters more than it sounds. With 5 DoF, there are orientations the gripper simply can't reach. The 6th joint gives you a free wrist, which for a task like reaching in at an angle to grab a cabinet door is the difference between "possible" and "constantly fighting the robot." The flip side: every degree of freedom you add is one more thing the model has to learn to coordinate.

This is me teleoperating a Seeed Studio follower arm with a leader arm:

<video class="img-large" autoplay loop muted playsinline>
  <source src="/assets/images/posts/imitation-learning/teleoperation.mp4" type="video/mp4">
</video>


Data collection is repetitive and a little tedious. You do the same task over and over, a few dozen times, trying to be consistent but also varied enough that the model doesn't memorize one exact trajectory. You quickly develop a deep respect for anyone who collects thousands of episodes.

And here's where I learned my lesson the hard way.

<div class="story-bar">
Last December, I was at a hackathon with a team. We spent the whole first day collecting demonstrations and training a model to do a manipulation task. By the end of the day it worked beautifully. We were thrilled. <br>
<br>
We came back the next morning, ran the exact same model... and it was a disaster. The robot fumbled, missed, did nonsense. Nothing about the code or the model had changed overnight. <br>
<br>
After a confused hour of debugging, we figured it out. One of my teammates, who had been sitting in the background of the camera frame during all of yesterday's recordings, had simply sat down in a different spot. The model had quietly learned that "my teammate is sitting *there*" was part of what the task looked like. Move him, and the whole thing fell apart. <br>
<br>
We still managed to give the judges a very nice demo, by having my teammate sit back exactly where he had been sitting during teleoperation.
</div>

We laughed about it at the time. But that story is actually the perfect doorway into the single most important concept in imitation learning, and the thing every model in this space is fighting against. Let me first show you the model I used, and then come back to *why* my teammate broke it.

## Under the hood: the ACT model

The model I trained from scratch for these tasks is called **ACT**, which stands for **Action Chunking with Transformers**. It came out of the {% include link_prominent.html url="https://tonyzhaozh.github.io/aloha/" text="ALOHA project" %} and is one of the most popular starting points for robot imitation learning today.

Let's build it up piece by piece, because every part of it is a fix for a specific problem.

### The problem: predicting one action at a time is fragile

The naive behavior cloning we described earlier predicts a single action from a single observation. Then the robot moves, observes again, predicts the next action, and so on.

The problem is that tiny errors **compound**. The robot makes a small mistake, which puts it in a slightly unfamiliar state, where it makes a slightly bigger mistake, which puts it somewhere even more unfamiliar... and within a second or two it has drifted somewhere it has never seen and has no idea what to do. (We'll formalize this in the next section, it has a name.)

### The fix: action chunking

Instead of predicting one action, ACT predicts a whole **chunk** of future actions at once, say the next ~100 steps.

![Action chunking](/assets/images/posts/imitation-learning/action_chunking.svg){: .img-large}

Why does this help? Because it shortens the number of decisions the model has to make. If you commit to 100 actions at a time, you've effectively divided the length of your task by 100, and there are far fewer opportunities for errors to accumulate and snowball. The robot follows a short, coherent plan instead of nervously re-deciding everything at every single step.

### Smoothing it out: temporal ensembling

There's a subtlety. If you predict a chunk, execute all of it, then predict the next chunk, you get a little "jerk" at the boundary between chunks where the new plan disagrees with the old one.

ACT's trick is **temporal ensembling**. It actually queries the model at *every* timestep, producing overlapping chunks, and then averages the predictions for the current moment across all the chunks that have an opinion about it. The result is smooth, continuous motion instead of jerky stop-and-go. (Notice the philosophy is the opposite of my pendulum, where I *penalized* jerky motion in the reward. Here there's no reward to do that, so the smoothing is baked into the architecture.)

### The clever bit: the CVAE and human messiness

Here's the part I find most elegant.

When a human demonstrates a task multiple times, the demonstrations are **multimodal**. You don't do the exact same thing every time. Sometimes you approach the cabinet handle from slightly above, sometimes from the side. Both are correct.

A naive network trained to minimize error on this data does something terrible: it averages the two valid approaches and aims for the middle, which might be straight into an obstacle. Averaging two correct answers can give you a very wrong one.

![Averaging two valid demonstrations](/assets/images/posts/imitation-learning/multimodal_averaging.svg){: .img-large}

ACT solves this by being a **CVAE** (Conditional Variational Autoencoder). The intuition, without drowning in the math:

- During training, an encoder looks at the demonstration and compresses its "style" into a small latent variable $z$. Think of $z$ as capturing *which way* you happened to do it this time.
- The main network (a transformer encoder-decoder) then generates the action chunk conditioned on the images, the joint positions, **and** this style variable $z$.
- The model is trained with two objectives: an $L_1$ loss that says "reproduce the demonstrated actions," and a KL-divergence term that keeps the space of styles $z$ well-behaved (close to a simple Gaussian).

$$
L = \underbrace{\| a - \hat{a} \|_1}_{\text{reproduce the actions}} + \beta \underbrace{D_{\text{KL}}\big(q(z \mid a, s) \,\|\, \mathcal{N}(0, I)\big)}_{\text{keep the style space tidy}}
$$

At test time, you simply set $z = 0$ (the "average style") and let the network produce a coherent action chunk. Because the multimodality was absorbed into $z$ during training, the network no longer has to average incompatible behaviors. It picks one clean way to do the task instead of mushing them together.

![ACT architecture](/assets/images/posts/imitation-learning/act_architecture.svg){: .img-large}

Put it all together: ResNet backbones turn the camera images into features, a transformer fuses those with the joint positions and the latent $z$, and out comes a chunk of smooth, human-like actions. Trained on a few dozen of my teleoperated episodes, this is what learned to open the cabinet.

## The Achilles heel: distribution shift

Now let's come back to my hackathon teammate.

The fundamental weakness of behavior cloning has a name: **distribution shift** (or covariate shift). It's the formal version of the compounding-errors problem we hand-waved at earlier.

Here's the issue. The model only ever saw the states that appear in your demonstrations. As long as it stays on that familiar path, it knows what to do. But the moment it drifts even slightly off-distribution, into a state it never saw in training, it's flying blind. And because each mistake pushes it further from familiar territory, errors don't just add up, they **compound**.

![Distribution shift](/assets/images/posts/imitation-learning/distribution_shift.svg){: .img-large}

Action chunking helps a lot here, because committing to a short plan gives errors fewer chances to snowball. But it doesn't *cure* the problem. Nothing fully does, short of collecting more and better data.

And this is exactly what happened at the hackathon. The model didn't actually learn "open the cabinet." It learned to reproduce the pixels it saw during training, and my teammate sitting in the background was one of those pixels. When he moved, the input no longer matched anything in the training distribution, and the robot was instantly off-distribution, flying blind. It hadn't learned the *task*. It had learned the *data*.

<div class="callout">
This is the fundamental bargain of imitation learning. It's wonderfully easy to train, because it's just supervised learning. But your model is only ever as good, and as general, as the demonstrations you give it. It imitates everything in your data, including the things you didn't mean to teach it.
</div>

So how do we get past this? How do we get robots that actually understand the task, that generalize to a cabinet they've never seen, in a kitchen they've never been in, with a teammate sitting wherever he likes? That's the frontier, and it's where things get really exciting.

## Scaling up: from one task to generalist robots

The ACT model I trained does exactly one thing: open one cabinet. Train it on a new task and you start more or less from scratch. That doesn't scale. You can't collect demonstrations for every task a robot might ever face.

The big idea on the frontier is the same one that transformed language and vision: **scale**. Instead of one small model per task, train one enormous model on a huge, diverse pile of robot data, across many tasks, many robots, and many environments. With enough variety, the model stops memorizing specific trajectories and starts learning something closer to general manipulation skills.

These models are called **VLAs**: **Vision-Language-Action** models.

![Vision Language Action model](/assets/images/posts/imitation-learning/vla.svg){: .img-large}

The name says it all. A VLA fuses three things into a single model:
- **Vision**: what the robot sees through its cameras.
- **Language**: an instruction in plain English, like *"open the electrical cabinet."*
- **Action**: the joint commands sent to the robot.

The "language" part is the game-changer. These models are typically built on top of a pre-trained **vision-language model** (the same kind of model that can look at a photo and describe it). That backbone already knows a staggering amount about the world from internet-scale data: what a cabinet is, what a handle is, what "open" means. The robot doesn't have to learn all of that from your forty demonstrations. It just has to connect that existing knowledge to motor actions.

### Fine-tuning Pi0.5

The VLA I got my hands on is **Pi0.5** (π0.5), from {% include link_prominent.html url="https://www.physicalintelligence.company/blog/pi05" text="Physical Intelligence" %}. It's one of the most capable open generalist robot policies available, explicitly designed for **open-world generalization**, the ability to operate in homes and environments it has never seen.

The contrast with ACT is striking:
- Training **ACT** from scratch, the model knows nothing. Your few dozen episodes have to teach it the entire task from zero.
- Fine-tuning **Pi0.5**, the model already understands cabinets, doors, handles, and the general physics of manipulation. Your episodes just nudge a knowledgeable generalist toward your specific setup. You need far less data to get something that generalizes far better.

One more technical nugget worth knowing, because it's a genuinely different approach from ACT's CVAE. To generate its continuous actions, Pi0.5 uses an **action expert** trained with **flow matching**.

You can think of flow matching as a cousin of the diffusion models behind AI image generators. Instead of producing an action in one shot, the model starts from pure random noise and learns a "velocity field" that smoothly *flows* that noise, over a few steps, into a real action chunk. The beautiful thing is that this handles the multimodality problem we discussed earlier very naturally: just like an image generator can produce many different valid images, flow matching can produce many different valid ways to do the task, without averaging them into mush. It's solving the same problem ACT's latent $z$ solves, but with a much more powerful and scalable tool.

![Flow matching intuition](/assets/images/posts/imitation-learning/flow_matching.svg){: .img-large}

## Imitation learning vs. reinforcement learning

Now that we've been through both this article and the {% include link_prominent.html url="https://nrdrgz.github.io/2025/12/02/double-pendulum-rl-deep-dive/" text="RL deep dive" %}, we can put them side by side honestly.

| | Reinforcement Learning | Imitation Learning |
|---|---|---|
| **What you provide** | A reward function | Demonstrations |
| **Needs a simulator?** | Usually yes | No, you can train on real data |
| **How it learns** | Trial and error, exploration | Copying examples |
| **Can discover new behavior?** | Yes (remember the pendulum swing-up I couldn't have demonstrated) | No, it only imitates what it's shown |
| **Main weakness** | Reward design and sim-to-real gap | Distribution shift, only as good as your data |
| **Great for** | Dynamic, hard-to-demonstrate skills (locomotion, balancing) | Manipulation, anything a human can easily demonstrate |

Neither one is "better." They're tools for different jobs. RL shines when the behavior is hard to demonstrate but easy to score (how would you teleoperate a robot into balancing my chaotic double pendulum?). Imitation shines when the behavior is easy to demonstrate but hard to score (what's the reward function for "open a cabinet nicely"?).

<div class="callout">
The most exciting part is that the future probably isn't one or the other, it's <strong>both</strong>. You can use imitation learning to bootstrap a sensible starting policy from demonstrations, then use reinforcement learning to refine it and make it robust, going beyond what a human ever showed it. Imitation gets you to "competent" fast and RL pushes you to "superhuman."
</div>

## What's next: toward robots that imagine

So we've now seen two of the big paradigms in modern robot learning. We *tell* a robot what good looks like with rewards, or we *show* it what to do with demonstrations. Both have a frustrating limitation in common, though: they need an enormous amount of experience, either millions of simulation steps or piles of demonstrations, because the robot has no real *model* of how the world works. It can't think ahead.

What if a robot could build its own internal simulator of reality, watch a situation, and *imagine* what would happen if it tried something, all in its head, before moving a single motor?

That's the idea behind **world models**, and it might be the thing that ties this whole series together: from rewards, to demonstrations, to imagination. We'll write an article about these.

<div class="callout">
Imitation learning is one of the most accessible ways to get into modern robotics. You don't need a reward function or a simulator, just an arm, a camera, and the patience to show it what to do a few dozen times. With tools like {% include link_prominent.html url="https://github.com/huggingface/lerobot" text="LeRobot" %} and affordable arms like the SO-101 and the Seeed Studio arm, there's never been a better time to teach a robot something yourself. Just keep an eye on who's sitting in the background.
</div>

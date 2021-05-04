# Intro to OpenAI Gym
In this Project, I will explore a reinforcement learning agent training platform OpenAI Gym, and Python RL library PyTorch AgentNet      
* You can see documentation and more info here: https://gym.openai.com/     
* You can see their source code and mode details how it works in their git: https://github.com/openai/gym.
* PyTorch DQN Tutorial: https://pytorch.org/tutorials/intermediate/reinforcement_q_learning.html
* The most current version of this notebook will be at the following GitHub Repo: https://github.com/ZZanetti/Atari_RL_Agent


## High-level overview

The Atari 2600 Freeway game is a good test of RL concepts. The agent has to decide which action to take - up, down, or switch chick, [0,1,2], and which obstacles to avoid, [cars]. I will use a DQN to train the agent. For background on DQN's, please read the Google Paper here - https://applied.cs.colorado.edu/pluginfile.php/23395/mod_resource/content/1/MnihEtAlHassibis15NatureControlDeepRL.pdf
. Our Deep Q-Learning model will use a CNN to process a series of game states and find the appropriate action to take. The network takes np arrays corresponding to pixels as input and outputs a set of Q values for each action. The action with the highest Q value will be taken by the agent. In order to perfect our model, the agent must do some exploration. In DQN's the agent will cache experiences and use memory recall to update Q-values. Unfortunately I was not able to replicate Google's 10,000 episodes of training due to storage contraints, which made training for games like Skiing-V0, and Assault-V0 difficult, however, I was able to achieve some training success on lower level games. I chose to present my findings with the Freeway game because it is a good display of how much an agent can learn in less than 100 episodes of training (with batch of 32).

## What does the game look like?

Enivronment: Freeway V0

States: Represented by an array of pixels [210,160,3]

Actions: [0,1,2]

Reward: Maximize reward by: Cross the Freeway without being hit by a car



```python
import gc
gc.collect()
```




    67




```python
import gym
import math
import numpy as np
import matplotlib
import matplotlib.pyplot as plt
from collections import namedtuple, deque
from itertools import count
from PIL import Image

import torch
import torch.nn as nn
import torch.optim as optim
import torch.nn.functional as F
import torchvision.transforms as T
from pathlib import Path
import random, datetime, os, copy

from gym.spaces import Box
from gym.wrappers import FrameStack

```


```python
gym.__version__
```




    '0.17.3'




```python
game_string = "Freeway-v0"
env = gym.make(game_string).unwrapped
env.reset()
next_state, reward, done, info = env.step(0)
print(f"{next_state.shape},\n {reward},\n {done},\n {info}")
```

    (210, 160, 3),
     0.0,
     False,
     {'ale.lives': 0}



```python
print(env.action_space)
```

    Discrete(3)


## Preprocessing

There are a few important preprocessing steps to take before inputting states from the environment into our CNN. Each state is a 260, 180, 3 array of pixels. Extracting information through convolutions in our model will take less time if we can reduce the dimensions of the array without losing any important information. 
* In the case of the Freeway game, the color of the cars/other objects is not important. If, for example, the chicken was supposed to walk through red cars, but not blue cars, it would be important to retain some color information, however, in this case we will use a GreyScaleConversion function to reduce the array to 1 channel instead of 3.

* We will also reduce the overall size of the image, as is done in the DQN paper. We can still retain the same information with less pixels, so we reduce the size to 84, 84. 
* I will implement skip frame function so the gym module automatically repeats the actions for 2-4 frames. (come back to this
* Lastly, I will implement a concept known as frame stack. From the PyTorch Docs: 
```
FrameStack is a wrapper that allows us to squash consecutive frames of the environment into a single observation point to feed to our learning model. This way, we can identify if Mario was landing or jumping based on the direction of his movement in the previous several frames
```
Essentially frame stack will take two consecutive states and extract transition information for the agent. 


```python
#Preprocess
# https://pytorch.org/tutorials/intermediate/mario_rl_tutorial.html
class SkipFrame(gym.Wrapper):
    def __init__(self, env, skip):
        """Return only every `skip`-th frame"""
        super().__init__(env)
        self._skip = skip

    def step(self, action):
        """Repeat action, and sum reward"""
        total_reward = 0.0
        done = False
        for i in range(self._skip):
            # Accumulate reward and repeat the same action
            obs, reward, done, info = self.env.step(action)
            total_reward += reward
            if done:
                break
        return obs, total_reward, done, info


class GrayScaleObservation(gym.ObservationWrapper):
    def __init__(self, env):
        super().__init__(env)
        obs_shape = self.observation_space.shape[:2]
        self.observation_space = Box(low=0, high=255, shape=obs_shape, dtype=np.uint8)

    def permute_orientation(self, observation):
        # permute [H, W, C] array to [C, H, W] tensor
        observation = np.transpose(observation, (2, 0, 1))
        observation = torch.tensor(observation.copy(), dtype=torch.float)
        return observation

    def observation(self, observation):
        observation = self.permute_orientation(observation)
        transform = T.Grayscale()
        observation = transform(observation)
        return observation


class ResizeObservation(gym.ObservationWrapper):
    def __init__(self, env, shape):
        super().__init__(env)
        if isinstance(shape, int):
            self.shape = (shape, shape)
        else:
            self.shape = tuple(shape)

        obs_shape = self.shape + self.observation_space.shape[2:]
        self.observation_space = Box(low=0, high=255, shape=obs_shape, dtype=np.uint8)

    def observation(self, observation):
      #should this be 0,1 or 0,255?
        transforms = T.Compose(
            [T.Resize(self.shape), T.Normalize(0, 255)]
        )
        observation = transforms(observation).squeeze(0)
        return observation


# Apply Wrappers to environment
env = SkipFrame(env, skip=4)
env = GrayScaleObservation(env)
env = ResizeObservation(env, shape=84)
env = FrameStack(env, num_stack=4)
```

## Our Model

#### Agent

The agent has three actions in a DQN - 

* Act - Take a random action or follow current highest Q value (exploration vs exploitation. Our agent will act based off of a declining learning rate. 

* Cache/Recall - Store the results of taking action i at state s and the result r, s'. Recall results later for use in learning.

* Learn - Update current action policy based on sample recakked of experiences.



```python
#https://pytorch.org/tutorials/intermediate/mario_rl_tutorial.html
class Chick:
    def __init__(self, state_dim, action_dim, save_dir):
        #INIT - this is where we can make tweaks to our model
        self.state_dim = state_dim
        self.action_dim = action_dim
        self.save_dir = save_dir

        self.use_cuda = torch.cuda.is_available()

        # DNN to predict the most optimal action - we implement this in the Learn section
        self.net = ChickNet(self.state_dim, self.action_dim).float()
        if self.use_cuda:
            self.net = self.net.to(device="cuda")

        self.exploration_rate = 1
        self.exploration_rate_decay = 0.9999975
        self.exploration_rate_min = 0.1
        self.curr_step = 0

        self.save_every = 1e4  # no. of experiences between saving chickNet
        #recall and cache
        self.memory = deque(maxlen=400000)
        #batch of 32 broke colab
        self.batch_size = 32

        # TD estimate/target
        self.gamma = 0.9

        #Q update
        #self.optimizer = torch.optim.RMSprop(self.net.parameters(), lr=0.00025, weight_decay=0.95, eps=self.exploration_rate_min)
        self.optimizer = torch.optim.Adam(self.net.parameters(), lr=0.00025)
        self.loss_fn = torch.nn.SmoothL1Loss()

        #learn
        #adjusted burnin for more exploration before training, updatd learn every to 4
        self.burnin = 10000 # min. experiences before training
        self.learn_every = 4  # no. of experiences between updates to Q_online
        self.sync_every = 1e4  # no. of experiences between Q_target & Q_online sync

    def act(self, state):
        """
    Given a state, choose an epsilon-greedy action and update value of step.

    Inputs:
    state(LazyFrame): A single observation of the current state, dimension is (state_dim)
    Outputs:
    action_idx (int): An integer representing which action chick will perform
    """
        # EXPLORE
        if np.random.rand() < self.exploration_rate:
            action_idx = np.random.randint(self.action_dim)

        # EXPLOIT
        else:
            state = state.__array__()
            if self.use_cuda:
                state = torch.tensor(state).cuda()
            else:
                state = torch.tensor(state)
            state = state.unsqueeze(0)
            action_values = self.net(state, model="online")
            action_idx = torch.argmax(action_values, axis=1).item()

        # decrease exploration_rate
        self.exploration_rate *= self.exploration_rate_decay
        self.exploration_rate = max(self.exploration_rate_min, self.exploration_rate)

        # increment step
        self.curr_step += 1
        return action_idx

    def cache(self, state, next_state, action, reward, done):
        """Add the experience to memory
         Inputs:
        state (LazyFrame),
        next_state (LazyFrame),
        action (int),
        reward (float),
        done(bool))
        """
        state = state.__array__()
        next_state = next_state.__array__()

        if self.use_cuda:
            state = torch.tensor(state).cuda()
            next_state = torch.tensor(next_state).cuda()
            action = torch.tensor([action]).cuda()
            reward = torch.tensor([reward]).cuda()
            done = torch.tensor([done]).cuda()
        else:
            state = torch.tensor(state)
            next_state = torch.tensor(next_state)
            action = torch.tensor([action])
            reward = torch.tensor([reward])
            done = torch.tensor([done])

        self.memory.append((state, next_state, action, reward, done,))

    def recall(self):
        """Sample experiences from memory"""
        batch = random.sample(self.memory, self.batch_size)
        state, next_state, action, reward, done = map(torch.stack, zip(*batch))
        return state, next_state, action.squeeze(), reward.squeeze(), done.squeeze()

    def learn(self):
        """Update online action value (Q) function with a batch of experiences"""
        if self.curr_step % self.sync_every == 0:
            self.sync_Q_target()

        if self.curr_step % self.save_every == 0:
            self.save()

        if self.curr_step < self.burnin:
            return None, None

        if self.curr_step % self.learn_every != 0:
            return None, None

        # Sample from memory
        state, next_state, action, reward, done = self.recall()

        # Get TD Estimate
        td_est = self.td_estimate(state, action)

        # Get TD Target
        td_tgt = self.td_target(reward, next_state, done)

        # Backpropagate loss through Q_online
        loss = self.update_Q_online(td_est, td_tgt)

        return (td_est.mean().item(), loss)
    
    def td_estimate(self, state, action):
        current_Q = self.net(state, model="online")[
            np.arange(0, self.batch_size), action
        ]  # Q_online(s,a)
        return current_Q

    @torch.no_grad()
    def td_target(self, reward, next_state, done):
        next_state_Q = self.net(next_state, model="online")
        best_action = torch.argmax(next_state_Q, axis=1)
        next_Q = self.net(next_state, model="target")[
            np.arange(0, self.batch_size), best_action
        ]
        return (reward + (1 - done.float()) * self.gamma * next_Q).float()

    def update_Q_online(self, td_estimate, td_target):
        loss = self.loss_fn(td_estimate, td_target)
        self.optimizer.zero_grad()
        loss.backward()
        self.optimizer.step()
        return loss.item()

    def sync_Q_target(self):
        self.net.target.load_state_dict(self.net.online.state_dict())

    def save(self):
        save_path = (
            self.save_dir / f"chick_net_{int(self.curr_step // self.save_every)}.chkpt"
        )
        torch.save(
            dict(model=self.net.state_dict(), exploration_rate=self.exploration_rate),
            save_path,
        )
        print(f"ChickNet saved to {save_path} at step {self.curr_step}")
```

#### Q Network

The Q Network is a CNN that will take in our preprocessed state, represented by two arrays of pixels in greyscale, with size 84,84,1, and values 0-255. I chose to use similar features to the Google DQN. Starting with a larger 8x8 kernel with 4,4 strides, and using smaller kernels and strides for each layer, while increasing the number of filters. The network will have a flattened layer with n=512, which will directly feed the output layer of n=3, given that we have 3 possible actions. 


```python
#https://applied.cs.colorado.edu/pluginfile.php/23395/mod_resource/content/1/MnihEtAlHassibis15NatureControlDeepRL.pdf
#https://pytorch.org/tutorials/intermediate/mario_rl_tutorial.html
class ChickNet(nn.Module):
    """mini cnn structure
  input -> (conv2d + relu) x 3 -> flatten -> (dense + relu) x 2 -> output
  """

    def __init__(self, input_dim, output_dim):
        super().__init__()
        c, h, w = input_dim

        if h != 84:
            raise ValueError(f"Expecting input height: 84, got: {h}")
        if w != 84:
            raise ValueError(f"Expecting input width: 84, got: {w}")

        self.online = nn.Sequential(
            nn.Conv2d(in_channels=c, out_channels=32, kernel_size=8, stride=4),
            nn.ReLU(),
            nn.Conv2d(in_channels=32, out_channels=64, kernel_size=4, stride=2),
            nn.ReLU(),
            nn.Conv2d(in_channels=64, out_channels=64, kernel_size=3, stride=1),
            nn.ReLU(),
            nn.Flatten(),
            nn.Linear(3136, 512),
            nn.ReLU(),
            nn.Linear(512, output_dim),
        )

        self.target = copy.deepcopy(self.online)

        # Q_target parameters are frozen.
        for p in self.target.parameters():
            p.requires_grad = False

    def forward(self, input, model):
        if model == "online":
            return self.online(input)
        elif model == "target":
            return self.target(input)
```

## Logging


```python
# https://pytorch.org/tutorials/intermediate/mario_rl_tutorial.html
import numpy as np
import time, datetime
import matplotlib.pyplot as plt


class MetricLogger:
    def __init__(self, save_dir):
        self.save_log = save_dir / "log"
        with open(self.save_log, "w") as f:
            f.write(
                f"{'Episode':>8}{'Step':>8}{'Epsilon':>10}{'MeanReward':>15}"
                f"{'MeanLength':>15}{'MeanLoss':>15}{'MeanQValue':>15}"
                f"{'TimeDelta':>15}{'Time':>20}\n"
            )
        self.ep_rewards_plot = save_dir / "reward_plot.jpg"
        self.ep_lengths_plot = save_dir / "length_plot.jpg"
        self.ep_avg_losses_plot = save_dir / "loss_plot.jpg"
        self.ep_avg_qs_plot = save_dir / "q_plot.jpg"

        # History metrics
        self.ep_rewards = []
        self.ep_lengths = []
        self.ep_avg_losses = []
        self.ep_avg_qs = []

        # Moving averages, added for every call to record()
        self.moving_avg_ep_rewards = []
        self.moving_avg_ep_lengths = []
        self.moving_avg_ep_avg_losses = []
        self.moving_avg_ep_avg_qs = []

        # Current episode metric
        self.init_episode()

        # Timing
        self.record_time = time.time()

    def log_step(self, reward, loss, q):
        self.curr_ep_reward += reward
        self.curr_ep_length += 1
        if loss:
            self.curr_ep_loss += loss
            self.curr_ep_q += q
            self.curr_ep_loss_length += 1

    def log_episode(self):
        "Mark end of episode"
        self.ep_rewards.append(self.curr_ep_reward)
        self.ep_lengths.append(self.curr_ep_length)
        if self.curr_ep_loss_length == 0:
            ep_avg_loss = 0
            ep_avg_q = 0
        else:
            ep_avg_loss = np.round(self.curr_ep_loss / self.curr_ep_loss_length, 5)
            ep_avg_q = np.round(self.curr_ep_q / self.curr_ep_loss_length, 5)
        self.ep_avg_losses.append(ep_avg_loss)
        self.ep_avg_qs.append(ep_avg_q)

        self.init_episode()

    def init_episode(self):
        self.curr_ep_reward = 0.0
        self.curr_ep_length = 0
        self.curr_ep_loss = 0.0
        self.curr_ep_q = 0.0
        self.curr_ep_loss_length = 0

    def record(self, episode, epsilon, step):
        mean_ep_reward = np.round(np.mean(self.ep_rewards[-100:]), 3)
        mean_ep_length = np.round(np.mean(self.ep_lengths[-100:]), 3)
        mean_ep_loss = np.round(np.mean(self.ep_avg_losses[-100:]), 3)
        mean_ep_q = np.round(np.mean(self.ep_avg_qs[-100:]), 3)
        self.moving_avg_ep_rewards.append(mean_ep_reward)
        self.moving_avg_ep_lengths.append(mean_ep_length)
        self.moving_avg_ep_avg_losses.append(mean_ep_loss)
        self.moving_avg_ep_avg_qs.append(mean_ep_q)

        last_record_time = self.record_time
        self.record_time = time.time()
        time_since_last_record = np.round(self.record_time - last_record_time, 3)

        print(
            f"Episode {episode} - "
            f"Step {step} - "
            f"Epsilon {epsilon} - "
            f"Mean Reward {mean_ep_reward} - "
            f"Mean Length {mean_ep_length} - "
            f"Mean Loss {mean_ep_loss} - "
            f"Mean Q Value {mean_ep_q} - "
            f"Time Delta {time_since_last_record} - "
            f"Time {datetime.datetime.now().strftime('%Y-%m-%dT%H:%M:%S')}"
        )

        with open(self.save_log, "a") as f:
            f.write(
                f"{episode:8d}{step:8d}{epsilon:10.3f}"
                f"{mean_ep_reward:15.3f}{mean_ep_length:15.3f}{mean_ep_loss:15.3f}{mean_ep_q:15.3f}"
                f"{time_since_last_record:15.3f}"
                f"{datetime.datetime.now().strftime('%Y-%m-%dT%H:%M:%S'):>20}\n"
            )

        for metric in ["ep_rewards", "ep_lengths", "ep_avg_losses", "ep_avg_qs"]:
            plt.plot(getattr(self, f"moving_avg_{metric}"))
            plt.savefig(getattr(self, f"{metric}_plot"))
            plt.clf()
```

## Training 

Training was a constant struggle with memory errors and model adjustments. My initial model was crafted from a tutorial that used the game super mario and built to play the Skiing Atari game. Unfortunately this model did not translate very well to the Skiing game. The Slkiing game has three actions where mario was limited to 2 (right and right-jump). The Skiing game was difficult to crack because moving left or right multiple times in a row actually makes you accelerate. On top of that, the gym module automatically skips 2-4 frames randomly. This means it is hard for the network to understand acceleration from multiple commands in a row. Because of the differences I spent a lot of time tweaking different rates, playing with different preprocessing, and configuring Google colab. Eventually I found a setup that could train without encountering a memory error. Once I had this going, I started tweaking variables like frameskip rate, batch size, learning decay, gamma, which optimizer, learn_every rate, and sync_every rate. Unfortunately I could not perform enough training episodes to make the agent any good at skiing. I decided to switch games to see if my model was totally useless or could be salvaged. With a few tweaks, I had achieved great results in the Freeway game.

![image.png](data:image/png;base64,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)


```python
#https://pytorch.org/tutorials/intermediate/mario_rl_tutorial.html
use_cuda = torch.cuda.is_available()
print(f"Using CUDA: {use_cuda}")
print()


save_dir = Path("checkpoints") / datetime.datetime.now().strftime("%Y-%m-%dT%H-%M-%S")
save_dir.mkdir(parents=True)

chick = Chick(state_dim=(4, 84, 84), action_dim=env.action_space.n, save_dir=save_dir)

logger = MetricLogger(save_dir)

episodes = 100
for e in range(episodes):

    state = env.reset()
    num_steps = 0
    # Play the game!
    while True:

        # Run agent on the state
        action = chick.act(state)

        # Agent performs action
        next_state, reward, done, info = env.step(action)
        num_steps += 1

        # Remember
        chick.cache(state, next_state, action, reward, done)

        # Learn
        q, loss = chick.learn()

        # Logging
        logger.log_step(reward, loss, q)

        # Update state
        state = next_state

        # Check if end of game
        if done:
            break

    logger.log_episode()

    if e % 5 == 0:
        logger.record(episode=e, epsilon=chick.exploration_rate, step=chick.curr_step)
```

    Using CUDA: True
    
    Episode 0 - Step 685 - Epsilon 0.9982889633544773 - Mean Reward 0.0 - Mean Length 685.0 - Mean Loss 0.0 - Mean Q Value 0.0 - Time Delta 3.071 - Time 2021-05-04T00:06:01
    Episode 5 - Step 4093 - Epsilon 0.989819661259441 - Mean Reward 0.0 - Mean Length 682.167 - Mean Loss 0.0 - Mean Q Value 0.0 - Time Delta 14.101 - Time 2021-05-04T00:06:15
    Episode 10 - Step 7518 - Epsilon 0.9813805015740439 - Mean Reward 0.091 - Mean Length 683.455 - Mean Loss 0.0 - Mean Q Value 0.0 - Time Delta 14.326 - Time 2021-05-04T00:06:30
    ChickNet saved to checkpoints/2021-05-04T00-05-52/chick_net_1.chkpt at step 10000
    Episode 15 - Step 10933 - Epsilon 0.9730376194664877 - Mean Reward 0.062 - Mean Length 683.312 - Mean Loss 0.0 - Mean Q Value 0.002 - Time Delta 15.976 - Time 2021-05-04T00:06:46
    Episode 20 - Step 14341 - Epsilon 0.9647825451827801 - Mean Reward 0.095 - Mean Length 682.905 - Mean Loss 0.0 - Mean Q Value 0.005 - Time Delta 18.063 - Time 2021-05-04T00:07:04
    Episode 25 - Step 17763 - Epsilon 0.9565640250800463 - Mean Reward 0.077 - Mean Length 683.192 - Mean Loss 0.0 - Mean Q Value 0.007 - Time Delta 18.642 - Time 2021-05-04T00:07:22
    ChickNet saved to checkpoints/2021-05-04T00-05-52/chick_net_2.chkpt at step 20000
    Episode 30 - Step 21174 - Epsilon 0.9484415964289832 - Mean Reward 0.129 - Mean Length 683.032 - Mean Loss 0.0 - Mean Q Value 0.008 - Time Delta 18.609 - Time 2021-05-04T00:07:41
    Episode 35 - Step 24588 - Epsilon 0.9403810844939412 - Mean Reward 0.194 - Mean Length 683.0 - Mean Loss 0.0 - Mean Q Value 0.01 - Time Delta 18.825 - Time 2021-05-04T00:08:00
    Episode 40 - Step 27983 - Epsilon 0.9324333659464334 - Mean Reward 0.293 - Mean Length 682.512 - Mean Loss 0.0 - Mean Q Value 0.012 - Time Delta 18.445 - Time 2021-05-04T00:08:18
    ChickNet saved to checkpoints/2021-05-04T00-05-52/chick_net_3.chkpt at step 30000
    Episode 45 - Step 31395 - Epsilon 0.9245135255754615 - Mean Reward 0.283 - Mean Length 682.5 - Mean Loss 0.0 - Mean Q Value 0.013 - Time Delta 18.299 - Time 2021-05-04T00:08:37
    Episode 50 - Step 34813 - Epsilon 0.916647204388353 - Mean Reward 0.333 - Mean Length 682.608 - Mean Loss 0.0 - Mean Q Value 0.014 - Time Delta 18.154 - Time 2021-05-04T00:08:55
    Episode 55 - Step 38246 - Epsilon 0.9088137334310462 - Mean Reward 0.411 - Mean Length 682.964 - Mean Loss 0.0 - Mean Q Value 0.015 - Time Delta 18.361 - Time 2021-05-04T00:09:13
    ChickNet saved to checkpoints/2021-05-04T00-05-52/chick_net_4.chkpt at step 40000
    Episode 60 - Step 41663 - Epsilon 0.9010832482952565 - Mean Reward 0.41 - Mean Length 683.0 - Mean Loss 0.0 - Mean Q Value 0.015 - Time Delta 18.286 - Time 2021-05-04T00:09:31
    Episode 65 - Step 45082 - Epsilon 0.893414052561634 - Mean Reward 0.439 - Mean Length 683.061 - Mean Loss 0.0 - Mean Q Value 0.016 - Time Delta 18.393 - Time 2021-05-04T00:09:50
    Episode 70 - Step 48505 - Epsilon 0.8858012719334234 - Mean Reward 0.408 - Mean Length 683.169 - Mean Loss 0.0 - Mean Q Value 0.016 - Time Delta 18.306 - Time 2021-05-04T00:10:08
    ChickNet saved to checkpoints/2021-05-04T00-05-52/chick_net_5.chkpt at step 50000
    Episode 75 - Step 51929 - Epsilon 0.8782511641711742 - Mean Reward 0.421 - Mean Length 683.276 - Mean Loss 0.0 - Mean Q Value 0.017 - Time Delta 18.57 - Time 2021-05-04T00:10:27
    Episode 80 - Step 55343 - Epsilon 0.8707871790218259 - Mean Reward 0.481 - Mean Length 683.247 - Mean Loss 0.0 - Mean Q Value 0.017 - Time Delta 18.478 - Time 2021-05-04T00:10:45
    Episode 85 - Step 58766 - Epsilon 0.8633672019700449 - Mean Reward 0.547 - Mean Length 683.326 - Mean Loss 0.0 - Mean Q Value 0.017 - Time Delta 18.186 - Time 2021-05-04T00:11:03
    ChickNet saved to checkpoints/2021-05-04T00-05-52/chick_net_6.chkpt at step 60000
    Episode 90 - Step 62184 - Epsilon 0.856021150749337 - Mean Reward 0.615 - Mean Length 683.341 - Mean Loss 0.0 - Mean Q Value 0.018 - Time Delta 18.351 - Time 2021-05-04T00:11:22
    Episode 95 - Step 65604 - Epsilon 0.8487333605002646 - Mean Reward 0.677 - Mean Length 683.375 - Mean Loss 0.0 - Mean Q Value 0.018 - Time Delta 18.446 - Time 2021-05-04T00:11:40



    <Figure size 432x288 with 0 Axes>


## Results 

You can see the mean reward growing in the log the above cell as the episodes increase. The agent is clearly getting better and better at finding the reward as epsilon decreases (randomness degree of actions)<br>
Overall, I failed with my original goal of training an agent to be good at the Skiing game, but succeeded at the Freeway game. I think I would have been better of using the Keras RL framework given my familiarity with Keras from HW5. I ended up using the PyTorch docs examples for a DQN and a DDQN with cartpole and mario as examples. As I discussed in the training section, the model did not translate well, due to things like acceleration on multiple action attemps and the randomness of action continuity ([2,3,4] actions in a row randomly). I was able to get improvement over a random agent in most cases, with my model scoring around -14K every run and the random agent scoring 16-19K every run. However, I was unable to train for 10,000 episodes due to CUDA memory restrictions in Colab. I think with more training the model would be sound. Because of poor results, I switched Atari games to something a little bit more simple. <br>
In the following cells I compare my agent with a random agent on 100 game iterations. I then provide a video of the random agent and my agent for reference. 


```python
!pip install gym pyvirtualdisplay > /dev/null 2>&1
!apt-get install -y xvfb python-opengl ffmpeg > /dev/null 2>&1
```


```python
import io
import glob
import base64
from IPython.display import HTML
from IPython import display as ipythondisplay

from pyvirtualdisplay import Display
display = Display(visible=0, size=(1400, 900))
display.start()
```




    <pyvirtualdisplay.display.Display at 0x7f1d1f495f50>




```python
#Geena Kim - Cartpole Edited
def show_video():
  mp4list = glob.glob('video/*.mp4')
  if len(mp4list) > 0:
    mp4 = mp4list[0]
    video = io.open(mp4, 'r+b').read()
    encoded = base64.b64encode(video)
    ipythondisplay.display(HTML(data='''<video alt="test" autoplay 
                loop controls style="height: 400px;">
                <source src="data:video/mp4;base64,{0}" type="video/mp4" />
             </video>'''.format(encoded.decode('ascii'))))
  else: 
    print("Could not find video")
    

def wrap_env(env):
  env = wrappers.Monitor(env, './video', force=True)
  return env
```


```python
#sample
from gym import wrappers
env = gym.make(game_string)

env = SkipFrame(env, skip=4)
env = GrayScaleObservation(env)
env = ResizeObservation(env, shape=84)
env = FrameStack(env, num_stack=4)
env = wrap_env(env)
Total = 0
runs = 100
for i in range(runs):
  total_reward = 0.0
  total_steps = 0
  obs = env.reset()

  while True:
    action = chick.act(state)
    obs, reward, done, _ = env.step(action)
    total_reward += reward
    total_steps += 1
    if done:
        break
  print("Episode %d done in %d steps, total reward %.2f" % (i, total_steps, total_reward))
  Total += total_reward
print("total: ", Total)
print("average: ", Total/runs)
```

    Episode 0 done in 682 steps, total reward 2.00
    Episode 1 done in 682 steps, total reward 2.00
    Episode 2 done in 680 steps, total reward 2.00
    Episode 3 done in 687 steps, total reward 2.00
    Episode 4 done in 683 steps, total reward 2.00
    Episode 5 done in 678 steps, total reward 1.00
    Episode 6 done in 688 steps, total reward 1.00
    Episode 7 done in 684 steps, total reward 1.00
    Episode 8 done in 682 steps, total reward 1.00
    Episode 9 done in 686 steps, total reward 2.00
    Episode 10 done in 677 steps, total reward 0.00
    Episode 11 done in 690 steps, total reward 1.00
    Episode 12 done in 685 steps, total reward 3.00
    Episode 13 done in 683 steps, total reward 0.00
    Episode 14 done in 672 steps, total reward 0.00
    Episode 15 done in 681 steps, total reward 5.00
    Episode 16 done in 685 steps, total reward 2.00
    Episode 17 done in 682 steps, total reward 1.00
    Episode 18 done in 683 steps, total reward 1.00
    Episode 19 done in 690 steps, total reward 1.00
    Episode 20 done in 685 steps, total reward 2.00
    Episode 21 done in 683 steps, total reward 1.00
    Episode 22 done in 677 steps, total reward 3.00
    Episode 23 done in 688 steps, total reward 1.00
    Episode 24 done in 684 steps, total reward 2.00
    Episode 25 done in 686 steps, total reward 4.00
    Episode 26 done in 686 steps, total reward 3.00
    Episode 27 done in 685 steps, total reward 2.00
    Episode 28 done in 687 steps, total reward 5.00
    Episode 29 done in 684 steps, total reward 1.00
    Episode 30 done in 679 steps, total reward 4.00
    Episode 31 done in 680 steps, total reward 4.00
    Episode 32 done in 684 steps, total reward 3.00
    Episode 33 done in 682 steps, total reward 2.00
    Episode 34 done in 677 steps, total reward 2.00
    Episode 35 done in 680 steps, total reward 2.00
    Episode 36 done in 680 steps, total reward 2.00
    Episode 37 done in 682 steps, total reward 3.00
    Episode 38 done in 678 steps, total reward 4.00
    Episode 39 done in 685 steps, total reward 2.00
    Episode 40 done in 681 steps, total reward 2.00
    Episode 41 done in 679 steps, total reward 3.00
    Episode 42 done in 683 steps, total reward 2.00
    Episode 43 done in 680 steps, total reward 2.00
    Episode 44 done in 686 steps, total reward 5.00
    Episode 45 done in 684 steps, total reward 2.00
    Episode 46 done in 679 steps, total reward 5.00
    Episode 47 done in 683 steps, total reward 4.00
    Episode 48 done in 679 steps, total reward 3.00
    Episode 49 done in 681 steps, total reward 4.00
    Episode 50 done in 678 steps, total reward 0.00
    Episode 51 done in 686 steps, total reward 3.00
    Episode 52 done in 677 steps, total reward 6.00
    Episode 53 done in 684 steps, total reward 3.00
    Episode 54 done in 679 steps, total reward 2.00
    Episode 55 done in 684 steps, total reward 5.00
    Episode 56 done in 685 steps, total reward 3.00
    Episode 57 done in 683 steps, total reward 5.00
    Episode 58 done in 678 steps, total reward 4.00
    Episode 59 done in 687 steps, total reward 5.00
    Episode 60 done in 686 steps, total reward 3.00
    Episode 61 done in 683 steps, total reward 4.00
    Episode 62 done in 684 steps, total reward 2.00
    Episode 63 done in 691 steps, total reward 4.00
    Episode 64 done in 689 steps, total reward 4.00
    Episode 65 done in 686 steps, total reward 6.00
    Episode 66 done in 675 steps, total reward 4.00
    Episode 67 done in 685 steps, total reward 2.00
    Episode 68 done in 682 steps, total reward 3.00
    Episode 69 done in 685 steps, total reward 2.00
    Episode 70 done in 690 steps, total reward 3.00
    Episode 71 done in 682 steps, total reward 5.00
    Episode 72 done in 686 steps, total reward 4.00
    Episode 73 done in 684 steps, total reward 2.00
    Episode 74 done in 684 steps, total reward 4.00
    Episode 75 done in 681 steps, total reward 3.00
    Episode 76 done in 683 steps, total reward 6.00
    Episode 77 done in 686 steps, total reward 4.00
    Episode 78 done in 683 steps, total reward 4.00
    Episode 79 done in 682 steps, total reward 6.00
    Episode 80 done in 685 steps, total reward 4.00
    Episode 81 done in 681 steps, total reward 3.00
    Episode 82 done in 689 steps, total reward 3.00
    Episode 83 done in 682 steps, total reward 5.00
    Episode 84 done in 690 steps, total reward 5.00
    Episode 85 done in 678 steps, total reward 7.00
    Episode 86 done in 686 steps, total reward 3.00
    Episode 87 done in 685 steps, total reward 4.00
    Episode 88 done in 686 steps, total reward 4.00
    Episode 89 done in 683 steps, total reward 2.00
    Episode 90 done in 676 steps, total reward 4.00
    Episode 91 done in 681 steps, total reward 4.00
    Episode 92 done in 679 steps, total reward 6.00
    Episode 93 done in 686 steps, total reward 5.00
    Episode 94 done in 690 steps, total reward 5.00
    Episode 95 done in 682 steps, total reward 5.00
    Episode 96 done in 681 steps, total reward 6.00
    Episode 97 done in 681 steps, total reward 3.00
    Episode 98 done in 688 steps, total reward 5.00
    Episode 99 done in 684 steps, total reward 8.00
    total:  316.0
    average:  3.16



```python
#Now Compare with random actions
Total = 0
for i in range(runs):
  total_reward = 0.0
  total_steps = 0
  obs = env.reset()

  while True:
    action = env.action_space.sample()
    obs, reward, done, _ = env.step(action)
    total_reward += reward
    total_steps += 1
    if done:
        break
  print("Episode done in %d steps, total reward %.2f" % (total_steps, total_reward))
  Total += total_reward
print("total: ", Total)
print("average: ", Total/runs)
```

    Episode done in 693 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 683 steps, total reward 0.00
    Episode done in 685 steps, total reward 1.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 682 steps, total reward 1.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 679 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 685 steps, total reward 1.00
    Episode done in 683 steps, total reward 0.00
    Episode done in 683 steps, total reward 0.00
    Episode done in 672 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 676 steps, total reward 0.00
    Episode done in 683 steps, total reward 0.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 683 steps, total reward 1.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 678 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 679 steps, total reward 0.00
    Episode done in 677 steps, total reward 0.00
    Episode done in 685 steps, total reward 0.00
    Episode done in 678 steps, total reward 0.00
    Episode done in 678 steps, total reward 0.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 685 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 685 steps, total reward 0.00
    Episode done in 685 steps, total reward 1.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 683 steps, total reward 0.00
    Episode done in 679 steps, total reward 1.00
    Episode done in 683 steps, total reward 0.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 673 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 686 steps, total reward 1.00
    Episode done in 682 steps, total reward 1.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 677 steps, total reward 0.00
    Episode done in 679 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 685 steps, total reward 0.00
    Episode done in 685 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 679 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 678 steps, total reward 0.00
    Episode done in 687 steps, total reward 0.00
    Episode done in 690 steps, total reward 0.00
    Episode done in 688 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 678 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 688 steps, total reward 0.00
    Episode done in 687 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 688 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 681 steps, total reward 0.00
    Episode done in 687 steps, total reward 1.00
    Episode done in 678 steps, total reward 1.00
    Episode done in 689 steps, total reward 0.00
    Episode done in 688 steps, total reward 0.00
    Episode done in 677 steps, total reward 1.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 680 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 683 steps, total reward 0.00
    Episode done in 677 steps, total reward 0.00
    Episode done in 678 steps, total reward 0.00
    Episode done in 687 steps, total reward 0.00
    Episode done in 686 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 684 steps, total reward 0.00
    Episode done in 681 steps, total reward 1.00
    Episode done in 682 steps, total reward 0.00
    Episode done in 681 steps, total reward 1.00
    Episode done in 685 steps, total reward 0.00
    total:  13.0
    average:  0.13


## Results (Continued)

There is a large difference in both the total score and average score of the ChickNet chick agent and the random agent. In 100 runs the Chick scored a total of 316 compared to the random agent's 13, for an average of 3.16 per attempt compared to 0.13 for the random agent.


```python
#Geena Kim - Cartpole Edited

env = gym.make(game_string)
#env = wrappers.Monitor(env, "recording", force=True) #force=True overrides the previously generated recording
chick = Chick(state_dim=(4, 84, 84), action_dim=env.action_space.n, save_dir=save_dir)
#env = SkipFrame(env, skip=2)
env = GrayScaleObservation(env)
env = ResizeObservation(env, shape=84)
env = FrameStack(env, num_stack=4)
env = wrap_env(env)

total_reward = 0.0
total_steps = 0
obs = env.reset()

while True:
    action = chick.act(state)
    obs, reward, done, _ = env.step(action)
    total_reward += reward
    total_steps += 1
    if done:
        break

print("Episode done in %d steps, total reward %.2f" % (total_steps, total_reward))
env.close()
show_video()
env.env.close()
```

    Episode done in 2731 steps, total reward 0.00



<video alt="test" autoplay 
                loop controls style="height: 400px;">
             </video>



```python
#Geena Kim - Cartpole Edited
#env = wrappers.Monitor(env, "recording", force=True) #force=True overrides the previously generated recording
total_reward = 0.0
total_steps = 0
obs = env.reset()

while True:
    action = env.action_space.sample()
    obs, reward, done, _ = env.step(action)
    total_reward += reward
    total_steps += 1
    if done:
        break

print("Episode done in %d steps, total reward %.2f" % (total_steps, total_reward))
env.close()
show_video()
env.env.close()
```

    Episode done in 2731 steps, total reward 0.00



<video alt="test" autoplay 
                loop controls style="height: 400px;">
             </video>


### References
* https://spinningup.openai.com/en/latest/spinningup/rl_intro.html
* https://github.com/Shmuma/ptan
* https://stackoverflow.com/questions/59789059/gpu-out-of-memory-error-message-on-google-colab
* https://gym.openai.com/envs/Freeway-v0/
* https://stackoverflow.com/questions/62303994/how-to-check-the-actions-available-in-openai-gym-environment
* https://pytorch.org/tutorials/intermediate/reinforcement_q_learning.html
* https://danieltakeshi.github.io/2016/11/25/frame-skipping-and-preprocessing-for-deep-q-networks-on-atari-2600-games/
* https://towardsdatascience.com/policy-gradient-methods-104c783251e0
* https://pytorch.org/tutorials/intermediate/mario_rl_tutorial.html
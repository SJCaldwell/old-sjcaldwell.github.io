# How to Roll Your Own OpenAI Gym Environment

1. TOC
{:toc}

In my [last post](/2020/04/28/deep-rl-infosec.html), I outlined my belief that reinforcement learning is the proper paradigm for creating an autonomous pentesting agent. In addition, I provided a brief [introduction to reinforcement learning](/2020/04/28/deep-rl-infosec.html#reinforcement-learning). Read that if you'd like a quick refresher, though for anyone looking to dive into the field properly I heartily recommend [Reinforcement Learning: An Introduction](https://web.stanford.edu/class/psych209/Readings/SuttonBartoIPRLBook2ndEd.pdf).

This blog post is part of my series on the development of an autonomous pentesting agent - if you don't know what that means and don't want to learn, feel free to skip to section 3 which goes over implementation. The code for this section can be found on my [github](https://github.com/SJCaldwell/multi-armed-bandit-gym).

In that post I discussed the multi-armed bandit, and wrote a little code to describe that as an environment. In reinforcement learning, an *environment* is what your agent interacts with, and how it recieves its reward after taking action. For ease of reference, I'll post the code from the last post below:

## A simple Environment

```python

import numpy as np

class Environment:
    def __init__(self, p):
        '''
        p is the probability of success for each 
        casino arm
        '''
        self.p = p
    
    def step(self, action):
        '''
        The agent pulls an arm and selects an action.
        The reward is stochastic - you only get anything 
        with the probability given in self.p for a given arm.
        
        action - the index of the arm you choose to pull
        '''
        result_prob = np.random.random() # Samples from continous uniform distribution
        if result_prob < self.p[action]:
            return 1
        else:
            return 0

p = [0.24, 0.33, 0.41]
env = Environment(p)
```

The above code instantiates a simple environment. An action must be `0`, `1`, or `2`. The probabilities of recieving a reward for each action are laid out as well. The API is similarly straightforward. It has only one `step` function, wherein you can take an action and it returns a result representing an observation: whether reward as recieved (`return 1`) or whether it was not (`return 0`).

This code represents a complete environment for the multi-armed bandit problem, and someone could technically plug an agent into this environment and attempt to solve it by recieving high reward. There's a proof-by-existence for that, in fact, since I did in the last post. 

However, my code has some disadvantages: for one, if we wanted to start a new *episode* for the algorithm with a fresh environment, there's no way to do so besides instantiating another Environment object with a different argument given to it. The API I chose is also arbitrary. Someone would need to read the code to interact with my environment. That's not a terrible thing because it's pretty simple, but if we designed a more complicated environment it would be hard to convince researchers to use it.

Reinforcement learning algorithms are very general - they can be applied to plenty different environments as long as an environment is properly specified and can reward an agent based on actions. This means the same 'agent' algorithms can be applied to plenty of situations. If there are plenty of agents that could interact with the environment, how would we define which one is best? Well, if you can interact with several kind of environments and "solve" them then your learning ability is likely very general. 

In a field as empirically driven as deep learning right now, benchmarks are an absolute necessity. The most well known example of these for vision is [ImageNet](http://www.image-net.org/). Every time a paper is published about a new architecture being developed, you know without reading the paper that the results section will feature reams of graphs comparing and contrasting the different architectures on the common task of ImageNet classification. It seems likely that without that common task to test algorithms against, reproducing the results of researchers and determining when a truly useful architecture had been created would be difficult and our progress might not be nearly as impressive. 

## Enter: OpenAI Gym

In 2016, OpenAI set out to solve the benchmarking problem and create something similar for deep reinforcement learning and developed the [OpenAI Gym](https://arxiv.org/pdf/1606.01540.pdf). The package provides several pre-built environments, and a web application shows off the leaderboards for various tasks. This makes measurement possible, and means comparing and contrasting agents is easy. In addition to these pre-built environments, they expose an interface to create your own custom environments.

Ultimately, the goal of creating an agent that can perform penetration testing currently is more environment than agent specific. If there is not a well-described environment for an agent to interact with, it's basically the same as having no training data in any other deep learning task. None of the cool deep learning models can help us without a well-described environment. For this reason, implementing the environment as a gym environment might seem unnecessary.

However, to make collaboration possible with other researchers, implementing the simulator with software easy for others to use will be important. This lowers the barrier to understanding. It also means any agent code I write can be benchmarked against the performance of other agents. Reinforcement learning is frustrating, and it's quite likely that if I write my own environment from scratch, *and* my own agent from scratch, something is going to be wrong in one of them. Being able to apply the agent code to known benchmarks will be incredibly useful in starting to debug whether it's my agent that can't learn, or my environment that can't *be* learned, or some painful combination of the two. 

Plus, I just think the OpenAI team is neat. Their paper on problems in AI safety is really inspired and thought provoking. AI safety is an important field, but it's easy to get overwhelmed with the feeling that there's no particular way to work on it: [Superintelligence](https://www.amazon.com/Superintelligence-Dangers-Strategies-Nick-Bostrom/dp/0198739834/) is a fun read, but would lead you to believe that any sufficiently complex algorithm will immediately start firing off nukes to make more paper clips to feed its inexhaustible desire for reward. And while regulation might play a role in AI safety as well, I'm definitely no legislator. But [OpenAI](https://arxiv.org/abs/1606.06565) turns some of these intangible, anxiety-ridden regulatory problems into something well-stated, technical, and ready to tackle. That's not to say that's the only way to solve the problem or that it's the only one that's important, it's just the one I'm most comfortable with. As discussed in the last post, making headway on some of these problems is also required to create an autonomous pentesting agent.

Anyway, that's enough preamble. Let's go on to writing our own gym!

## The Gym Interface

For starters, let's turn our crappy little environment above into an OpenAI gym. It's really simple and self-contained, so we'll start with that!

First, let's go over what the major methods are for the `Env`.

`reset(self):` Reset the environment's state. Returns observation. This is ideal for running a new episode without creating a new object in memory.

`step(self, action):` Step the environment by one timestep. Returns *observation*, *reward*, *done*, and *info*. 

An *observation* is what the agent can know about their environment at this time step. If you were playing a game, this might represent a frame of it.

The *reward* is pretty straightforward. This is the amount of reward you got for the last action. 

*done* is a boolean indicating whether the environment needs resetting. Most tasks have *episodes* when it's necessary to reset them. For example, if you were playing a video game, you might have died (or less morbidly, you won!).

*info* is a dictionary. This is all diagnostic information for debugging. For example, if your state transitions carry probabilities with you this would probably be where you put those. The agent can't see any of these. 

`render(self, mode='human)`: Render one frame of the environment. The default mode will do something human friendly, like create a pop up window. If this was Connect4, you might get an image of the state of the board. Same with chess. Good way to observe if your agent is interacting with the environment in a way that makes sense.

This is a little more in depth than what we wrote for the toy version of our environment, but it's probably obvious how useful many of these would be if our environment was less of a toy and more important. RL is notorious for being [difficult to debug]() so when we start implementing deep reinforcement learning algorithms we'll be glad for anything we can get beyond graphs of our flat reward functions.

### Basic Implementation

```python
import random
import gym
from gym import Env, logger, spaces
import numpy as np

np.random.seed(0)

class MultiArmedBanditEnv(Env):

    def __init__(self, n=3, info={}):
        """
        n - number of arms in the bandit
        """
        self.num_bandits = n
        self.action_space = spaces.Discrete(self.num_bandits)
        self.observation_space = spaces.Discrete(1) # just the reward of the last action
        self.bandit_success_prob = np.random.uniform(size=self.num_bandits) # Pick some random success probabilities
        self.info = info

```

We're going to keep this environment basic. Our `MultiArmedBanditEnv` will only take two arguments, which is `n` for the number of bandits with their probability of success, which we instantiate with `np.random.uniform(size=self.num_bandits)`. We also pass an info dictionary as we discussed before. This will act as a running scratch space if we want it. This formulation of the multi-armed bandit problem doesn't really evolve, so there's not really any good information to bother saving.We'll include it anyway, for correctness.

We also need to define our *action space* and *observation space*. There are many kinds of these, all of which are outlined in `gyms.spaces`. For this problem it's straightforward - our action space is discrete, equivalent to the number of bandits. Our observation space is a single value - basically no value. Some actions give us more reward than others, but nothing about the environment ever looks different in bandit land. We'll represent that as a list with only a 0 in it.

```python
    def step(self, action):
        reward = 0
        done = True
        result_prob = np.random.random()
        if result_prob < self.bandit_success_prob[action]:
            reward = 1
        else:
            reward = 0
        return [0], reward, done, self.info
```

Our step function has changed a bit to accomodate the gym interface. Our observation is set to `[0]` and will always be. The agent never "sees" more about the environment than the reward coming back from a chosen action. We set `done=True` since the environment will never evolve any further. We will assign reward essentially the same as before. 

```python
    def reset(self):
        # Get some new bandit success probs
        self.bandit_success_prob = np.random.uniform(size=self.num_bandits) # Pick some random success probabilities

    def render(self, mode='human'):
        print('bandits success prob:')
        for i in range(self.num_bandits):
            print("arm {num} reward prob: {prob}".format(num=i, prob=self.bandit_success_prob[i]))

```

For us, reset should just reroll some new bandit success probabilities. Render is also pretty simple - we just create a basic structure of printing those success probabilities. This would be used for comparing the agents conception of arm reward probability with the true distribution.


### Registering Environment
Alright, that's a good start, but we're not quite ready to run this environment yet. To make it possible to see the environment from within OpenAI's gym framework, it needs to register with the gym environment. To do that, it needs a name and a version. 

This is actually really easy! You just need to provide an *environment name*, *version*, *entry point* and *whether the environment is nondetermnistic*. 

We'll make our environment into a list of lists called `environments` to allow me to come back and make new bandit environments in the future. There are an incredibly number of formulation of the bandit problem that are all pretty interesting, and it would be need to offer them all in the same package.

The entry point code will make more sense if you see how the code is structured, and you can check that out [here](https://github.com/SJCaldwell/multi-armed-bandit-gym/tree/master/bandit_gym).

```python
from gym.envs.registration import register

from .multi_armed_bandit_env import MultiArmedBanditEnv

environments = [['MultiArmedBanditEnv', 'v0']]

for environment in environments:
    register(
        id='{}-{}'.format(environment[0], environment[1]),
        entry_point='bandit_gym:{}'.format(environment[0]),
        nondeterministic=True
    )
```

### Testing the Environment

Now we can install the environment. To test the code, after you've pip installed the library, simply run:

```python
import gym
import bandit_gym

env = gym.make("MultiArmedBanditEnv-v0")
env.reset()
for _ in range(1000):
    env.step(env.action_space.sample())
env.render()
env.close()
```

Your output should look something like:

```
bandits success prob:
arm 0 reward prob: 0.5448831829968969
arm 1 reward prob: 0.4236547993389047
arm 2 reward prob: 0.6458941130666561
```

If that worked, your installation was successful!

Now that we know the basics of the installation process, we'll make a more complicated environment in our next post!
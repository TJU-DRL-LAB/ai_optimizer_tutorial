
## User Guidance
Here we introduce how to configure your own dataset and modify the algorithm based on your own design. 

### Dataset
* Rewrite *tjuOfflineRL.get_dataset.py* to add *get_your_data* function in get_dataset function.
```
def get_dataset(
    env_name: str, create_mask: bool = False, mask_size: int = 1) -> Tuple[MDPDataset, gym.Env]:
  
    if env_name == "existing datasets":
        return get_existing_datasets()
    elif env_name == "your own datasets":
        return get_your_data()
    raise ValueError(f"Unrecognized env_name: {env_name}.")
```
* Load your datasets and transform then into *MDPDataset* format
```
def get_your_data():
    observations = []
    actions = []
    rewards = []
    terminals = []
    episode_terminals = []
    episode_step = 0
    cursor = 0
    dataset_size = dataset["observations"].shape[0]
    while cursor < dataset_size:
        # collect data for step=t
        observation = dataset["observations"][cursor]
        action = dataset["actions"][cursor]
        if episode_step == 0:
            reward = 0.0
        else:
            reward = dataset["rewards"][cursor - 1]

        observations.append(observation)
        actions.append(action)
        rewards.append(reward)
        terminals.append(0.0)

        # skip adding the last step when timeout
        if dataset["timeouts"][cursor]:
            episode_terminals.append(1.0)
            episode_step = 0
            cursor += 1
            continue

        episode_terminals.append(0.0)
        episode_step += 1

        if dataset["terminals"][cursor]:
            # collect data for step=t+1
            dummy_observation = observation.copy()
            dummy_action = action.copy()
            next_reward = dataset["rewards"][cursor]

            # the last observation is rarely used
            observations.append(dummy_observation)
            actions.append(dummy_action)
            rewards.append(next_reward)
            terminals.append(1.0)
            episode_terminals.append(1.0)
            episode_step = 0

        cursor += 1

    mdp_dataset = MDPDataset(
        observations=np.array(observations, dtype=np.float32),
        actions=np.array(actions, dtype=np.float32),
        rewards=np.array(rewards, dtype=np.float32),
        terminals=np.array(terminals, dtype=np.float32),
        episode_terminals=np.array(episode_terminals, dtype=np.float32),
        create_mask=create_mask,
        mask_size=mask_size,
    )

    return mdp_dataset, env

```
* get your own datasets by
```
from tjuOfflineRL.datasets import get_dataset

parser = argparse.ArgumentParser()
parser.add_argument('--dataset', type=str, default='your dataset')
args = parser.parse_args()
get_dataset(args.dataset)
```
### Modify Algorithm
Assuming you're modifying algorithm based on SAC: 
* Create two python file, name them as *YourSAC.py* and *YourSACImpl.py*. 其中*YourSACImpl.py*中指定的*YourSACImpl* class继承*SACImpl*.
```
class YourSACImpl(SACImpl):
    def __init__(self, a=A, b=B):
        ...
```
* Modify your algo in *YourSACImpl.py* by overloading *compute_critic_loss/compute_actor_loss/other* functions.
```
def compute_critic_loss(self, batch: TorchMiniBatch, q_tpn: torch.Tensor) -> torch.Tensor:
    observations = batch.observations
    actions = batch.actions
    rewards = batch.next_rewards
    ...
    your_critic_loss = critic_loss_func(observations, actions, rewards)
    return your_critic_loss

def compute_actor_loss(self, batch: TorchMiniBatch) -> torch.Tensor:
    observations = batch.observations
    actions = batch.actions
    ...
    your_actor_loss = actor_loss_func(observations, actions)
    return your_actor_loss
```
* Import *YourSACImpl* in *YourSAC.py* and modify *_create_impl* function to pass your algorithm parameters to *YourSACImpl.py*
```
def _create_impl(self, observation_shape: Sequence[int], action_size: int) -> None:
    self._impl = YourSACImpl(a=A, b=B, ...)
    self._impl.build()
```

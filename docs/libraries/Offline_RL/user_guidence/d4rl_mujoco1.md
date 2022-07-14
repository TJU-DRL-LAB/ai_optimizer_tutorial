
# D4RL MuJoCo
## Prepare Dataset


```
from tjuOfflineRL.datasets import get_d4rl

dataset, env = get_d4rl(dataset_name)
```
The potential ``dataset_name`` options of D4RL MuJoCo (v2) lie in the following list:
```
datasets = {
            'halfcheetah-expert-v2', 'hopper-expert-v2', 'walker2d-expert-v2', 'ant-expert-v2', 
            'halfcheetah-full-replay-v2', 'hopper-full-replay-v2', 'walker2d-full-replay-v2', 'ant-full-replay-v2', 
            'halfcheetah-medium-expert-v2', 'hopper-medium-expert-v2', 'walker2d-medium-expert-v2', 'ant-medium-expert-v2', 
            'halfcheetah-medium-replay-v2', 'hopper-medium-replay-v2', 'walker2d-medium-replay-v2', 'ant-medium-replay-v2', 
            'halfcheetah-medium-v2', 'hopper-medium-v2', 'walker2d-medium-v2', 'ant-medium-v2', 
            'halfcheetah-random-v2', 'hopper-random-v2', 'walker2d-random-v2', 'ant-random-v2', 
           }
```
The ``dataset`` can be slpitted into a training dataset and a test dataset for further analyzing.


```
from sklearn.model_selection import train_test_split

train_episodes, test_episodes = train_test_split(dataset, test_size=0.2)
```

## Setup Algorithm


D4RL MuJoCo consists a suit of continuos control tasks, thus the available algorithms include:
```
  offline_rl_algorithms = ['AWAC', 'AWR', 'BCQ', 'UWAC', 'BEAR', 'CRR', 'CQL',\
                           'MOPO', 'COMBO', 'PLAS', 'TD3PlusBC', 'SAC-N', 'EDAC']
  behaviour_cloning_algorithms = ['BC', 'ProbablisticBC']
  online_rl_algorithms = ['DDPG', 'SAC', 'TD3']
  other_algorithms = ['RandomPolicy']
```

Suppose the ``CQL`` algorithm is chosed: 
```
from tjuOfflineRL.algos import CQL

if "medium-v2" in args.dataset:
    conservative_weight = 10.0
else:
    conservative_weight = 5.0
    
encoder = tjuOfflineRL.models.encoders.VectorEncoderFactory([256, 256, 256])
 
cql = CQL(actor_learning_rate=1e-4,
         critic_learning_rate=3e-4,
         temp_learning_rate=1e-4,
         actor_encoder_factory=encoder,
         critic_encoder_factory=encoder,
         batch_size=256,
         n_action_samples=10, # action sample number in 
         alpha_learning_rate=0.0, #  if greater than 0, the the alpha is dynamically adjusted
         conservative_weight=conservative_weight,  # set the conservative_weight of conservative_loss.
         use_gpu=args.gpu  # if you don't use GPU, set use_gpu=False instead.
         ) 
```


## Setup Metrics

The metrics is computed through scikit-learn style scorer function and online evaluation.

```
from tjuOfflineRL.metrics.scorer import td_error_scorer
from tjuOfflineRL.metrics.scorer import average_value_estimation_scorer
from tjuOfflineRL.metrics.scorer import evaluate_on_environment

# calculate metrics with test dataset
td_error = td_error_scorer(dqn, test_episodes)

# set environment in scorer function
evaluate_scorer = evaluate_on_environment(env)

# evaluate algorithm on the environment
rewards = evaluate_scorer(dqn)
```



## Start Training

Now, you have all to start  training.

```
cql.fit(dataset.episodes,
        eval_episodes=test_episodes,
        n_steps=1000000, # total training steps
        n_steps_per_epoch=1000,
        save_interval=100, # save model per 100 epochs
        tensorboard_dir='cql_runs',
        scorers={
            'environment': tjuOfflineRL.metrics.evaluate_on_environment(env),
            'value_scale': tjuOfflineRL.metrics.average_value_estimation_scorer,
        },
        experiment_name=f"CQL_{args.dataset}_{args.seed}")
```
Then, you will see training progress in the console like below::

```
load datafile: 100%|██████████████████████████████| 9/9 [00:02<00:00,  4.05it/s]
2022-07-14 15:36.34 [debug    ] RandomIterator is selected.
2022-07-14 15:36.34 [info     ] Directory is created at tjuOfflineRL_logs/CQL_halfcheetah-random-v2_0_20220714153634
2022-07-14 15:36.34 [debug    ] Building models...
2022-07-14 15:36.38 [debug    ] Models have been built.
2022-07-14 15:36.38 [info     ] Parameters are saved to tjuOfflineRL_logs/CQL_halfcheetah-random-v2_0_20220714153634/params.json params={'action_scaler': None, 'actor_encoder_factory': {'type': 'vector', 'params': {'hidden_units': [256, 256, 256], 'activation': 'relu', 'use_batch_norm': False, 'dropout_rate': None, 'use_dense': False}}, 'actor_learning_rate': 0.0001, 'actor_optim_factory': {'optim_cls': 'Adam', 'betas': (0.9, 0.999), 'eps': 1e-08, 'weight_decay': 0, 'amsgrad': False}, 'alpha_learning_rate': 0.0, 'alpha_optim_factory': {'optim_cls': 'Adam', 'betas': (0.9, 0.999), 'eps': 1e-08, 'weight_decay': 0, 'amsgrad': False}, 'alpha_threshold': 10.0, 'batch_size': 256, 'conservative_weight': 5.0, 'critic_encoder_factory': {'type': 'vector', 'params': {'hidden_units': [256, 256, 256], 'activation': 'relu', 'use_batch_norm': False, 'dropout_rate': None, 'use_dense': False}}, 'critic_learning_rate': 0.0003, 'critic_optim_factory': {'optim_cls': 'Adam', 'betas': (0.9, 0.999), 'eps': 1e-08, 'weight_decay': 0, 'amsgrad': False}, 'gamma': 0.99, 'generated_maxlen': 100000, 'initial_alpha': 1.0, 'initial_temperature': 1.0, 'n_action_samples': 10, 'n_critics': 2, 'n_frames': 1, 'n_steps': 1, 'q_func_factory': {'type': 'mean', 'params': {'bootstrap': False, 'share_encoder': False}}, 'real_ratio': 1.0, 'reward_scaler': None, 'scaler': None, 'soft_q_backup': False, 'target_reduction_type': 'min', 'tau': 0.005, 'temp_learning_rate': 0.0001, 'temp_optim_factory': {'optim_cls': 'Adam', 'betas': (0.9, 0.999), 'eps': 1e-08, 'weight_decay': 0, 'amsgrad': False}, 'use_gpu': 0, 'algorithm': 'CQL', 'observation_shape': (17,), 'action_size': 6}
Epoch 1/1000: 100%|█| 1000/1000 [00:39<00:00, 25.20it/s, temp_loss=9.6, temp=0.9
2022-07-14 15:37.30 [info     ] CQL_halfcheetah-random-v2_0_20220714153634: epoch=1 step=1000 epoch=1 metrics={'time_sample_batch': 0.002517345190048218, 'time_algorithm_update': 0.03693196225166321, 'temp_loss': 9.598375787734986, 'temp': 0.9521856915354728, 'critic_loss': 28.63472083854675, 'actor_loss': -2.9735559566020964, 'time_step': 0.03957478475570679, 'environment': 2.24897581700692, 'value_scale': -1.5535942057561252} step=1000
.
.
.
```



## Save and Load

There are several ways to save trained models.

```
  # save full parameters
  cql.save_model('cql.pt')

  # load full parameters
  cql = CQL()
  cql.build_with_dataset(dataset)
  cql.load_model('cql.pt')

  # save the greedy-policy as TorchScript
  cql.save_policy('policy.pt')

  # save the greedy-policy as ONNX
  cql.save_policy('policy.onnx', as_onnx=True)
```
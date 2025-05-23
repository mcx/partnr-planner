# Adding a new planner

In this repo, Planners are classes which are responsible for outputting the low level actions for one or many agents at each time step of the simulation. All planners should inherit from the base class [Planner](/habitat_llm/planner/planner.py), and override methods as necessary. The only method which must be overridden is `get_next_action`, which takes three arguments: the natural language instruction for the current task, a dictionary of sensor observations, and the current world graph for all agents the planner must provide actions for.

We can easily add a new planner by extending the Planner class and overriding this method. Below is a planner which will use the existing [Rearrange Skill](/habitat_llm/tools/motor_skills/rearrange/oracle_rearrange_skill.py) to have each agent move random objects to random receptacles.

The full code for the class is

```python
import random
from typing import TYPE_CHECKING, Any, Dict, Tuple

from habitat_llm.llm.instruct.utils import get_objects_descr, get_world_descr
from habitat_llm.planner import Planner

if TYPE_CHECKING:
    from habitat_llm.agent.env import EnvironmentInterface
    from habitat_llm.world_model.world_graph import WorldGraph


class RandomRearrangePlanner(Planner):
    """
    A planner that randomly rearranges objects in the environment.
    """

    def __init__(self, env_interface: "EnvironmentInterface"):
        """
        Initialize the LLMPlanner.

        :param plan_config: The planner configuration.
        :param env_interface: The environment interface.
        """
        super().__init__(None, env_interface)
        self.last_high_level_actions = {}

    def reset(self):
        """
        Reset the planner state.
        """
        for agent in self._agents:
            agent.reset()
        self.last_high_level_actions = {agent_id: None for agent_id in self._agents}

    def get_next_action(
        self,
        instruction: str,
        observations: Dict[str, Any],
        world_graphs: Dict[int, "WorldGraph"],
        verbose: bool = False,
    ) -> Tuple[Dict[int, Any], Dict[str, Any], bool]:
        """
        Get the next low-level action to execute.

        :param instruction: The instruction for the task.
        :param observations: The current observations.
        :param world_graph: The world graph for each agent.
        :param verbose: Whether to print verbose output. Defaults to False.
        :return: A tuple containing:
                 - The low-level actions for each agent
                 - Planner information for logging
                 - Whether the planner is done
        """
        high_level_actions = {}
        # Loop through all agents to determine which skill to call
        for agent_id, world_graph in world_graphs.items():
            # If there was a high level action for this agent which has not completed, continue using it
            if self.last_high_level_actions.get(agent_id, None) is not None:
                high_level_actions[agent_id] = self.last_high_level_actions[agent_id]
            else:
                # If there was no high level action for this agent, or the last high level action has completed,
                # randomly select an object and receptacle to move to
                receptacles = world_graph.get_all_furnitures()
                objects = world_graph.get_all_objects()
                object_to_move = random.sample(objects, 1)[0].name
                receptacle_to_move_to = random.sample(receptacles, 1)[0].name
                # The format for the rearrange skill is:
                # Rearrange[<object_to_be_moved>, <spatial_relation>, <furniture_to_be_placed>, <spatial_constraint>, <reference_object>]
                # This agent will ignore spatial relations so we leave those parameters as None
                high_level_actions[agent_id] = (
                    "Rearrange",
                    f"{object_to_move}, on, {receptacle_to_move_to}, None, None",
                    None,
                )
                self.last_high_level_actions[agent_id] = high_level_actions[agent_id]
                print(f"Calling {high_level_actions[agent_id]} for agent {agent_id}")
        # Get low level actions and/or termination responses
        low_level_actions, responses = self.process_high_level_actions(
            high_level_actions, observations
        )
        for agent_id, response in responses.items():
            if len(response) > 0:
                # A non-empty response indicates that the skill has completed
                print(f"Response for agent {agent_id} skill: {response}")
                # Print out the current world graph
                print("Current world graph after skill execution:")
                print(get_objects_descr(world_graphs[agent_id],agent_uid=agent_id))
                # Reset the high level action for this agent
                self.last_high_level_actions[agent_id] = None
        return low_level_actions, {}, False
```
This file is included in the repo at [/habitat_llm/planner/random_rearrange_planner.py](/habitat_llm/planner/random_rearrange_planner.py).

In order to run this planner with an episode dataset first we create a hydra configuration corresponding to the new planner in the planner configuration folder [/habitat_llm/conf/planner/random_rearrange_planner.yaml](/habitat_llm/conf/planner/random_rearrange_planner.yaml).

```yaml
_target_: 'habitat_llm.planner.random_rearrange_planner.RandomRearrangePlanner'
_recursive_: True
_partial_: True
```

This configuration just points to class we just created so the planner can be instantiated by the evaluation runner.

Finally we need to create the hydra configuration for an evaluation using this planner. In order to have this planner plan for both agents we can extend the [example centralized planning example config](/habitat_llm/conf/examples/centralized_full_obs.yaml) and override only the planner.

```yaml
defaults:
  - /examples/centralized_full_obs
  - override /planner@evaluation.planner: random_rearrange_planner
```

If we place this file in [/habitat_llm/conf/examples/random_rearrange.yaml](/habitat_llm/conf/examples/random_rearrange.yaml) we can run our planner against the val set of episodes with

```bash
python -m habitat_llm.examples.planner_demo --config-name examples/random_rearrange.yaml habitat.dataset.data_path="data/datasets/partnr_episodes/v0_0/val_mini.json.gz"
```

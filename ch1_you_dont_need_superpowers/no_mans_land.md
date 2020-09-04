# No man's land

### Version 3: let's face it, no algorithm is enough!

Great! reko is almost there, the DP algorithm collects x units of utility. Reco have one last riddle to solve, in reality, each call has a probabilistic output. For example, if he spends 45 Mins with the client 1 there is 85% probability to succeed and 15% to fail and gain nothing, in the other hand if he spends only 15 Mins the client will not feel comfortable and the probability reduces to 35% of success and 65% of failure. This introduces two problems, first: what does it mean to find an optimal solution when the output is probabilistic. And second, even if we could solve it, from where on plant earth we could get these probabilities! Reco took a large cup of coffee and went on deep thinking, after a couple of hours Reco recognized that the algorithm should mutually optimize for total value and learn the probabilities on the way. Does the combo of optimization and learning sound familiar to you? of course, this is what bandits do.


> **_NOTE:_** it is common that constraints of real-world problems make the algorithm design a hard task. Instead, model the problem and let the solvers do the heavy lifting for you ;)

* Issue 1: How to overcome the pathological case of our algorithm
* Issue 2: How to account for the probabilistic outcome 
* Issue 3: How to solve the problem with missing info

the dynamic programming worked well for this example, however, there are few problems with it: first, for cases of small meeting time with longer slot available, the algorithm may pic the same slot multiple times (as in example below), and second what if the client is providing multiple available slots in his calendar, the same client may be picked for multiple meetings in one day. Although you can modify your algorithm to avoid these cases it is most likely not a good idea. In the real world scenarios, the problem will be modified in various ways that make the algorithm design a hard task and you will need to revamp your algorithm over and over.

For example, think about what will happen if you are asked to use the algorithm with a team of marketing agents instead of one, and what if each of them has different availability time and workload? these modifications will make the problem harder and most likely NP-Complete and there will not be a good algorithm that scale to more than a few marketing slots per calendar. In that case, to avoid being fired, you want to design an algorithm that results in good enough solutions instead of an optimal one.  

Instead of working hard on the algorithm, we want to work hard on modeling the problem and use a tool that comes up with good solutions, even in case the problem is NP-Complete. This will allow us to save the algorithm development time and make iterative solutions by changing the objective and adding constraints to our problem. Fortunately, for a large portion of such a problem, there are optimization methods that could be used.

```{.python .input  n=1}
# !pip install pyomo
# !pyomo install-extras
#!conda install -y -c conda-forge pyomo.extras
#!conda install -y -c conda-forge glpk
import random
import numpy as np
import pandas as pd
import pdb
import math
%matplotlib inline 
```

#### A sanity check baseline
So how to know that the solution is good enough after all. A good idea is to build some baseline to compare with because we don't care about the perfect solution the baseline should be simple enough. in this example, let's say we pick the highest value fist until we don't have space.

```{.python .input  n=2}
#code
```

the solution of the baseline is wrench, hammer, and towel which sum up to 14 the same as ... (TODO find better weights)

#### Scheduling as an optimization problem
Now, let's return to our scheduling problem. We define the optimization problem as follows:

- **Variables**: boolean variable for each possible slot to be picked or not.
- **Objective**: maximize the sum of value from scheduled meetings
- **Constraint**: sum(slots per client) <= 1 for each client (each client could be contacted at most once per day)
- **Constraint**: Sum(slots) <= 1 for each overlap

and pretty much that's it, now lets put it into code:

**1. Generate sample problems**: instead of using fixed problem we generate sample problems for testing our algorithms. for each client, we generate 1-2 available times per day, each of these could randomly start between 8 and 14 and have a random duration of 15, 30, 45 or 60 mins. And finally, each available time is associated with some random weight.

```{.python .input  n=3}
def generate_schedual(clients):
    schedual = []
    for client in clients:
        num_slots = random.randint(1, 2)
        for i in range(num_slots):
            start = float(random.randint(8, 14)) + random.randint(0, 4) * 0.25
            duration = 0.25 * random.randint(1, 4)
            end = start + duration
            weight = random.randint(1, 5)
            schedual += [[client, start, end, duration, weight]]
    return schedual
```

**2. Enumerate slots** each available time per client may have different solts, we enumerate them all and store them into a pandas data frame for convenience.

```{.python .input  n=4}
from collections import defaultdict
# tasks = [["T1", 08.50, 09.50, 1.00, 3], 
#          ["T2", 09.25, 10.00, 0.75, 4],
#          ["T3", 09.50, 10.75, 0.50, 2],
#          ["T4", 10.25, 11.50, 0.25, 1],
#          ["T5", 12.00, 13.50, 0.75, 1],
#          ["T6", 12.25, 13.75, 0.75, 2],
#          ["T1", 12.00, 13.50, 1.00, 3]]

tasks = generate_schedual(["T1", "T2", "T3", "T4", "T5", "T6"])
def find_possible_slots(start_time, end_time, duration):
    for i in range(int((end_time - start_time) / 0.25)):
        end_time_ = start_time + i * 0.25 + duration
        if end_time_ <= end_time:
            yield (start_time + i * 0.25, start_time + i * 0.25 + duration, duration)

def task_id(task, slot):
    return {"task": task[0] + "_" + str(slot[0]) + "_" + str(slot[1]), 
            "task_group": task[0], 
            "start": slot[0], 
            "finish": slot[1], 
            "duration": slot[2], 
            "weight": task[4]}

def compute_possible_tasks(tasks):
    tasks_ = []
    for task in tasks:
        slots = find_possible_slots(float(task[1]), float(task[2]), float(task[3]))
        for slot in slots:
            tasks_ += [task_id(task, slot)]
    tasks_  = sorted(tasks_, key=lambda x: x["finish"])
    return pd.DataFrame(tasks_)
```

**3. Find overlaps** We find overlapped tasks using pandas, please not this implementation is not efficient however we use it here for illustration purposes, in production you need to consider using better data structure like segment tree to find overlapping tasks efficiently.

```{.python .input  n=5}
def find_task_overlaps(possible_tasks):
    overlapping_tasks = {}
    for task in possible_tasks.task.unique():
        start, finish = possible_tasks[possible_tasks.task == task].iloc[0][["start", "finish"]].values
        overlapping_tasks_ = possible_tasks[(possible_tasks.start >= start) & (possible_tasks.start < finish)]
        overlapping_tasks_ = list(overlapping_tasks_.task.unique())
        if len(overlapping_tasks_) > 1:
            overlapping_tasks[task] = overlapping_tasks_
    return overlapping_tasks

```

**4. Optimize** we define the optimization problem. (break it down)

```{.python .input  n=8}
def schedual_tasks(tasks):
    model = ConcreteModel()

    #data
    possible_tasks = compute_possible_tasks(tasks)
    tasks = possible_tasks.task.values
    w = {r.task: r.weight for _, r in possible_tasks[possible_tasks.task == tasks].iterrows()}

    #constarint data
    overlapping_tasks = find_task_overlaps(possible_tasks)
    task_groups = possible_tasks["task_group"].unique()

    #variables
    model.x = Var(tasks, within=Binary )

    #objective
    model.value = Objective(expr = sum(w[i]*model.x[i] for i in tasks), sense = maximize )

    #constraints
    @model.Constraint(task_groups)
    def one_each_group(m, tg):
        return sum(m.x[task] for task in possible_tasks[possible_tasks["task_group"] == tg]["task"].unique()) <= 1

    @model.Constraint(overlapping_tasks.keys())
    def one_each_overlap(m, t):
        return sum(m.x[task] for task in overlapping_tasks[t]) <= 1

    #solve
    opt = SolverFactory('glpk')
    result_obj = opt.solve(model)
    selected = [k for k, v in model.x.get_values().items() if v == 1]
    
    #formate resutls
    results = (possible_tasks
          .loc[possible_tasks.task.isin(selected)]
          .sort_values(by=['start'])
          .set_index("task_group")
          [["start", "finish", "duration", "weight"]])
    return results

```

```{.python .input  n=14}
#results = schedual_tasks(tasks)
#results
```

```{.python .input  n=16}
#results.weight.sum()
```

**5. Baseline sanity check** to verify that our solution is working fine, we set a greedy algorithm that takes the most valuable tasks first and eliminates all conflicting tasks. 

```{.python .input  n=17}
def solve_by_elemination(tasks):
    schedual = []
    possible_tasks = compute_possible_tasks(tasks)
    possible_tasks_ = possible_tasks.copy().sort_values(by=['weight'], ascending=False)
    for i in range(100):
        try:
            task, task_group, start, finish = possible_tasks_.iloc[i][["task", "task_group", "start", "finish"]]
            possible_tasks_ = possible_tasks_[~((possible_tasks_.start >= start) 
                                                & (possible_tasks_.start < finish) 
                                                & ((possible_tasks_.task != task)))]
            possible_tasks_ = possible_tasks_[~((possible_tasks_.finish > start) 
                                                & (possible_tasks_.finish <= finish) 
                                                & ((possible_tasks_.task != task)))]
            possible_tasks_ = possible_tasks_[(possible_tasks_.task_group != task_group) 
                                              | (possible_tasks_.task == task ) ]
            schedual += [task]
        except:
            break
    return possible_tasks[possible_tasks.task.isin(schedual)]

solve_by_elemination(tasks).weight.sum()
```

```{.json .output n=17}
[
 {
  "data": {
   "text/plain": "20"
  },
  "execution_count": 17,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

In this case, we see that our optimization is surpassing the baseline and the results make sense.

```{.python .input}

```

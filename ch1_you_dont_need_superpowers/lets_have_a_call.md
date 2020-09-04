# Let's have a call

Shifting gears from problem solving to modeling give us the ability find solutions to complex problems that we even could not dream about solving them. Lets take step out and think about the bigger picture once more, instead of focusing on trip planning problem we can focus on building tip planner product which will allow us to add more valuable features to travelers. Unlike problems, products are living creatures, you need always to add new features, make impovements based on customers feedback, after all your travelers dont care about the details, they just want to have the best trip and you will be responsible for shaping the product, setting problems and solutions on there behalf. 

Lets imaging you are hired for an imaginary company, as the company data scientest the CEO asked you to build a phone call schedular for his marketing department. There are number of clients we need to market our poduct to, some of these clientes are more important than others, we need to set the optimal schedual that yeilds the best value and does not overload our agents.



### Version1: the activity selection problem
You strat reading about schedualing algorithms and decided to start simple, you will devide the clients into separate groups and guess one slot per day for each client. Each group will be handeled by one marketing aginet. The input is a set of tasks with start and end time, the tasks have conflicting periods and one person can only perform one task at a time. The goal is to select the maximum number of tasks that can be executed by one person. After conducting your research about sechdualing algorithms, you noticed similarity to the activity scheduling problem. 

Fortunatly the activity selection problem can be solved using a greedy solution, start with the task that finish first then pick consiquent tasks that finish first and does not conflict with the last task you have selected.

```{.python .input  n=1}
#TODO add drawing of the system
```

```{.python .input  n=2}
tasks = [("C1", 8.50, 9.50), 
         ("C2", 9.25, 10.25),
         ("C3", 9.30, 10.75),
         ("C4", 10.25, 11.50),
         ("C5", 12.00, 13.50),
         ("C6", 12.25, 13.75)]
```

```{.python .input  n=3}
# Algorithm: Activity selection problem
# inputs: tasks with start and finish times
# output: selected_tasks list of maximum none confilecting tasks

#1. stort data by finish time
tasks = sorted(tasks, key=lambda x: x[2])

#2. start with the taks that have the earliest finsh time
last_selected_task = tasks.pop(0)
last_selected_task_finish = last_selected_task[2]
selected_tasks = [last_selected_task]

#3. add tasks with earlist finish that does not conflict with the seleected tasks
while len(tasks) > 0:
    task = tasks.pop(0)
    task_start = task[1]
    if task_start >= last_selected_task_finish:
        last_selected_task_finish = task[2]
        selected_tasks += [task]


print (selected_tasks)
```

```{.json .output n=3}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "[('C1', 8.5, 9.5), ('C4', 10.25, 11.5), ('C5', 12.0, 13.5)]\n"
 }
]
```

```{.python .input  n=4}
import pandas as pd
from tabulate import tabulate
import math

df = pd.DataFrame(selected_tasks, columns=["task", "start time", "finish time"]).set_index("task")
print(tabulate(df, headers='keys', tablefmt='psql'))

```

```{.json .output n=4}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "+--------+--------------+---------------+\n| task   |   start time |   finish time |\n|--------+--------------+---------------|\n| C1     |         8.5  |           9.5 |\n| C4     |        10.25 |          11.5 |\n| C5     |        12    |          13.5 |\n+--------+--------------+---------------+\n"
 }
]
```

By applying the greedy algorithm on our example, your marketing agent can talk to 3 clients on that day with no conflicts. The greedy solution is optimal, to see that, let’s assume we have an optimal solution that does not start with the first finishing task, if we replace the first task in the solution with the task that is earliest finishing task (note that no time conflict here), then we will have another optimal solution. Now we need to select the optimal tasks from the remaining tasks, so we are solving the Activity selection problem again on small problem size, we can apply the same logic until the smaller problem reach size zero. 


### Version 2: Weighted activity selection problem
This solution seems to work fine, however, we could better model our problem. first, agents will not use all the time to make the call, for example, if the slot is 45 Mins, the agent may need only 30 Mis. After all, clients have different concerns and personalities, for easy-going clients, the agent could finish in 15 Mins and for other less pleasant clients, he may need up to an hour. Second, some clients are more valuable to the company, so the agents needs to take client value into consideration when schedualing for the phonecalls.

**Allow schedular to pick sub slot:**
To implement this featuer, you divid each time slot on 15 min bases, so if the client is free for 1 hour the schedular will consider 15, 30, 45 and 60 mins, the agent is required to enter the expected time for each client based on experiance. Note that, in this setting, you can start after the slot starting time, for example, if the client is free from 10:00 AM - 11:00 AM and he needs 45 mins, the schedular will consider two options, either 10:00 AM - 10:45 AM or 10:15 AM-11:00 AM.

<img src="./img/weighted tasks.PNG" width="600px"/>

Note that activity selection will , this problem can be modeled as an acyclic directed graph(DAG), and the solution of the problem is the longest path in that graph. he longest path in DAG is easy, they could be solved by dynamic programming (DP) algorithm.

<img src="./img/Constraint graph.PNG" width="600px"/>

In general, the longest paths are hard problems, actually, they are NP-Complete. However, fortunately,

```{.python .input  n=9}
from collections import defaultdict
tasks = [("T1", 8.30, 9.30, 0.30, 3), 
         ("T2", 9.15, 10.15, 0.45, 4),
         ("T3", 9.30, 10.45, 1.15, 2),
         ("T4", 10.15, 11.30, 0.15, 1),
         ("T5", 12.00, 13.30, 0.30, 5),
         ("T6", 12.15, 13.45, 0.45, 2)]

tasks = sorted(tasks, key=lambda x: x[2])

def base_60_to_base_10(t):
    t_hr, t_min = str(t).split(".")
    t_hr = float(t_hr)
    t_min = float(t_min)
    if t_min < 10:
        t_min = t_min * 10
    t = t_hr + (t_min/60.0)
    return t

def base_10_to_base_60(t):
    t_hr, t_min = str(t).split(".")
    t_hr = float(t_hr)
    t_min = float(t_min)
    if t_min < 10:
        t_min = t_min * 10
    t = t_hr + (t_min/100.0 * 0.6)
    return t


def find_possible_slots(start_time, end_time, duration):
    for i in range(int((end_time - start_time) / 0.25)):
        end_time_ = start_time + i * 0.25 + duration
        if end_time_ <= end_time:
            yield (start_time + i * 0.25, start_time + i * 0.25 + duration, duration)
            
            
def connect_tasks_to_successor(tasks, i):
    #tasks that will start after i finish and start <= finish of connected to task
    #we dont connect last task
    if i >= len(tasks) - 1:
        return
    
    to_connect = []
    task = tasks[i]
    max_successor_start = task["finish"]
    for task_ in tasks[i+1:]:
        if task["task_group"] == task_["task_group"]:
            continue
        if task_["start"] >= task["finish"] and (task_["start"] <= max_successor_start or len(to_connect) == 0):
            to_connect += [task_["task"]]
            max_successor_start = max(task_["finish"], max_successor_start)
            
    return to_connect


tasks_ = []
for task in tasks:
    slots = list(find_possible_slots(base_60_to_base_10(task[1]), base_60_to_base_10(task[2]), base_60_to_base_10(task[3])))
    for slot in slots:
        tasks_ += [{"task": task[0] + "_" + str(slot[0]) + "_" + str(slot[1]), "task_group": task[0], "start": slot[0], "finish": slot[1], "duration": slot[2], "weight": task[4]}]
tasks_  = sorted(tasks_, key=lambda x: x["finish"])

#build the DAG
DAG = []
for i in range(0, len(tasks_)):
    task = {"task": tasks_[i]["task"], "info": tasks_[i], "next_tasks": connect_tasks_to_successor(tasks_, i)}
    DAG += [task]
    
best_weight = defaultdict(float)
best_from = defaultdict(str)
weight = {task["task"]: task["info"]["duration"]  for task in DAG}

for i in DAG:
    if i["next_tasks"] is not None:
        for next_task in i["next_tasks"]:
            if weight[next_task] + best_weight[i["task"]] > best_weight[next_task]:
                best_weight[next_task] = weight[next_task] + best_weight[i["task"]]
                best_from[next_task] = i["task"]
                
last_task = sorted(best_weight.items(), key=lambda i:i[1])[-1][0]
path = [last_task]
while best_from[last_task] != "":
    path += [best_from[last_task]]
    last_task = best_from[last_task]
    
path

```

```{.json .output n=9}
[
 {
  "data": {
   "text/plain": "['T6_13.0_13.75',\n 'T5_12.0_12.5',\n 'T4_10.75_11.0',\n 'T3_9.5_10.75',\n 'T1_8.5_9.0']"
  },
  "execution_count": 9,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

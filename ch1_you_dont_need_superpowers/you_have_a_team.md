# You have a team

```{.python .input  n=7}
agents = ("Alex", "Jennifer", "Andrew", "DeAnna", "Jesse")

clients = (
    "Trista", "Meredith", "Aaron", "Bob", "Jillian",
    "Ali", "Ashley", "Emily", "Desiree", "Byron")
```

```{.python .input  n=27}
import numpy as np
import sklearn
from sklearn import datasets
import random
import math 

def score(agent, client, agents_v, clients_v):
    try:
        s = 1 / (1 + math.exp(-np.dot(agents_v[agent], clients_v[client])))
    except:
        print (np.dot(agents_v[agent], clients_v[client]))
        return random.random()
    return s
    
def generate_matching_problem(agents, clients):
    num_samples = len(agents) + len(clients)
    samples = sklearn.datasets.make_swiss_roll(num_samples, noise=3, random_state=0)[0]
    random.shuffle(samples)
    clients_v = {clients[i]: samples[i] for i in range(len(clients))}
    agents_v = {agents[i]: samples[i] for i in range(len(agents))}
    match_scores = dict(
        ((agent, client), score(agent, client, agents_v, clients_v))
        for agent in agents
        for client in clients)

    client_time = {client: random.randint(1, 4) for client in clients}
    agents_workload = {agent: random.randint(2, 5) for agent in agents}
    
    return match_scores, client_time, agents_workload, clients_v, agents_v
```

```{.python .input  n=35}
from pyomo.environ import *
import pyomo.environ as pe 

def solve_matching(match_scores, client_time, agents_workload):
    agents = agents_workload.keys()
    clients = client_time.keys()
    
    model = pe.ConcreteModel()
    model.agents = agents_workload.keys()
    model.clients = clients
    model.match_scores = match_scores
    model.agents_workload = agents_workload

    model.assignments = pe.Var(match_scores.keys(), domain=pe.Binary)
    model.objective = pe.Objective(
            expr=pe.summation(model.match_scores, model.assignments),
            sense=pe.maximize)

    @model.Constraint(model.agents)
    def respect_workload(model, agent):
        return sum(model.assignments[agent, client] * client_time[client] for client in model.clients) <= model.agents_workload[agent]

    @model.Constraint(model.clients)
    def one_agent_per_client(model, client):
        return sum(model.assignments[agent, client] for agent in model.agents) <= 1


    solver = pe.SolverFactory("glpk")
    solver.solve(model)
    sln = [k for k, v in model.assignments.get_values().items() if v == 1.0]
    return sln, sum(match_scores[i] for i in sln)
```

```{.python .input  n=36}
def solve_matching_greedy(match_scores, client_time, agents_workload):
    agents = agents_workload.keys()
    clients = client_time.keys()
    matching = sorted(match_scores.items(), key=lambda x: -x[1])
    clients_indicator = {client: 0 for client in clients}
    agents_workload_ = agents_workload.copy()
    sln = []
    for (agent, client), score in matching:
        if clients_indicator[client] == 0 and agents_workload_[agent] >= client_time[client]:
            clients_indicator[client] = 1
            agents_workload_[agent] -= client_time[client]
            sln += [(agent, client)]
    return sln, sum([match_scores[i] for i in sln])
```

```{.python .input  n=37}
match_scores, client_time, agents_workload, clients_v, agents_v = generate_matching_problem(agents, clients)
```

```{.python .input  n=38}
solve_matching(match_scores, client_time, agents_workload)
```

```{.json .output n=38}
[
 {
  "data": {
   "text/plain": "([('Andrew', 'Aaron'),\n  ('Jennifer', 'Byron'),\n  ('Jesse', 'Jillian'),\n  ('DeAnna', 'Trista'),\n  ('DeAnna', 'Meredith'),\n  ('Alex', 'Desiree'),\n  ('Alex', 'Bob'),\n  ('Jesse', 'Ali'),\n  ('Andrew', 'Ashley')],\n 9.0)"
  },
  "execution_count": 38,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

```{.python .input  n=39}
solve_matching_greedy(match_scores, client_time, agents_workload)
```

```{.json .output n=39}
[
 {
  "data": {
   "text/plain": "([('Alex', 'Trista'),\n  ('Alex', 'Meredith'),\n  ('Alex', 'Aaron'),\n  ('Jennifer', 'Bob'),\n  ('Andrew', 'Jillian'),\n  ('Andrew', 'Ali'),\n  ('DeAnna', 'Ashley'),\n  ('Jesse', 'Emily')],\n 8.0)"
  },
  "execution_count": 39,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

```{.python .input}

```

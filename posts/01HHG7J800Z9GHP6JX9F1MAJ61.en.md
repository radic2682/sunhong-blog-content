---
id: "01HHG7J800Z9GHP6JX9F1MAJ61"
title_en: "Power Generation and Transmission Planning: Objectives and Constraints"
summary_en: "A generation and transmission planning project from a power systems engineering course. It models new power plants and transmission expansion to meet long-term demand growth as a cost-minimization problem, and covers the constraints, power flow calculation, MILP optimization, and scenario analysis."
---
![Generation and Transmission Planning|large](/img/01HHG7J800Z9GHP6JX9F1MAJ61/10ee6f05e456.webp)

This is a **Generation and Transmission Planning** project from a power systems engineering course. The assignment was to build a plan for new power plants and transmission expansion to meet growing electricity demand over the long term.

## Project Background

Power systems are planned and built over years to decades. In the short term there is the Economic Dispatch problem of adjusting the output of existing plants, but in the long term you have to answer questions like these:

- Where should new plants be built, and of what type?
- Where should transmission lines be built?
- How do you guarantee reliability under a budget constraint?
- How do you meet environmental regulations and renewable energy targets?

The goal of this project was to model this long-term planning problem as a mathematical optimization problem and solve it.

## Generation Planning

### Objective Function
- Total cost = investment cost of new plants + fuel cost (operating cost over time) + maintenance cost
- Minimize this cost subject to the constraints

### Key Variables
- Capacity of each plant (MW)
- Generation type (coal, LNG, nuclear, wind, solar, etc.)
- Construction timing (the year it comes online)

### Constraints
- **Demand coverage**: supply must exceed hourly and annual electricity demand
- **Reserve margin**: keep spare capacity at a set ratio above peak demand
- **Utilization limits per source**: nuclear runs at fixed output, renewables are intermittent
- **Emission regulation**: a cap on CO2 emissions
- **Budget constraint**: an investment limit per year

## Transmission Planning

### Purpose
Plan the transmission lines that reliably deliver power from the plants to the load centers. This is tightly coupled with generation planning.

### Key Considerations
- **Transmission capacity**: maximum transferable power per line (MVA)
- **Line losses**: minimize I²R losses
- **Contingency handling**: the N-1 criterion (the whole system keeps operating normally even if one line fails)
- **Construction cost**: cost differences by terrain, distance, and voltage class

### Power Flow Calculation
This calculates how power will actually flow through the planned network. I used the DC Power Flow approximation to simplify it into a linear problem:

```
B × θ = P
```

Here B is the susceptance matrix of the network, θ is the phase angle at each bus, and P is the net injected power at each bus.

## Optimization Method

### Mixed-Integer Linear Programming (MILP)
Whether to build a plant (0 or 1) was modeled as a binary variable and generation output as a continuous variable, and the result was solved as a MILP problem. One of the following tools was used:

- **Python + PuLP / Pyomo**: free, good for learning
- **MATLAB + YALMIP + CPLEX/Gurobi**: widely used in industry
- **GAMS**: the standard in economics and power engineering

### Scenario Analysis
Beyond deterministic optimization, I also ran a scenario analysis that accounts for uncertainty in demand growth and fuel prices. Comparing the plans produced under each scenario identified a robust plan.

## Results and Analysis

Key results from the project:
- **Base load**: a combination of nuclear and coal covers the base demand
- **Intermediate load**: LNG combined-cycle plants respond flexibly
- **Peak load**: pumped-storage hydro and demand response
- **Renewables**: gradual growth in wind and solar, but backup generation is required
- **Transmission network**: stronger links between renewable-heavy regions and load centers

## Lessons Learned

- **Complexity of power planning**: it is not just mathematical optimization; policy, environmental, and social factors are intertwined. It is not a purely technical problem
- **Time scales**: a layered structure of second-level control (ACE), minute-level adjustment (AGC), hourly economic dispatch, daily generation scheduling, and yearly capacity planning
- **The renewable challenge**: intermittency and unpredictability make renewables worth less per unit of capacity. ESS or flexible generation is essential
- **Limits of optimization tools**: there is a gap between the ideal mathematical model and real operation. Approximations and empirical tuning are needed
- **Economy vs. reliability**: a fundamental trade-off between the two axes. Safe plans are expensive, and cheap plans are risky

This project let me see how power systems engineering theory (Kirchhoff's laws, power flow, economic dispatch) carries over into long-term planning in real industry.

# well-architected

## Tenets

- Stop guessing your capacity needs
- Test systems at production scale
- Automate to make your architectural experiments easier
- Allow for evolutionary architectures
- Drive architectures using data
- Improve through game days

## Pillars

### Operational Excellence

#### Principles

- Perform operations as code
- Annotate documentation
  - Use annotations as input to operations code??
- Make frequent, small, reversible changes
- Refine operations procedures frequently
- Anticipate failure
- Learn from all operational failures

#### Best practice areas

- Prepare
  - Key Service:  AWS Config
  - **OPS 1**:  How do you determine what your priorities are?
  - **OPS 2**:  How do you design your workload so that you can understand its state?
  - **OPS 3**:  How do you reduce defects, ease remediation, and improve flow into production
  - **OPS 4**:  How do you mitigate deployment risks?A
  - **OPS 5**:  How do you know that you are ready to support a workload?
- Operate
  - Key Service:  Amazon CloudWatch
  - **OPS 6**:  How do you understand the health of your workload?
  - **OPS 7**:  How do you understand the health of your operations?
  - **OPS 8**:  How do you manage workload and operations events?
- Evolve
  - Key Service:  Amazon Elasticsearch Service
  - **OPS 9**:  How do you evolve operations?

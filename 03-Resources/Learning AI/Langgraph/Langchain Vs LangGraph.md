draw backs of langchain 
1. Control Flow: conditional control flow, jump, loops are not possible in langchain without Glue code
2. State management: Langchain is stateless framework so to manage State we need to manually handle the states and pass it on increasing potential fault points
3. Event Driven Execution: in Langchain event driven execution is not possible and in langgraph it is possible to pause and resume whole workflow at any pointy of the time
4. Fault tolerance: as langchain is made up for single run small linear execution, is state less and doesn't have retry mechanism or checkpoint so it is not possible to resume any workflow from where it stopped
5. Human in the loop: Langgraph has HTL as a first class citizen so at any point the workflow can be pause and only resumed after we recieve event from human
6. Nested workflows: In Lang Graph we can make nested workflow
7. Observability: Using langsmith we can monitor both langchain and langgraph but  langsmith only monitors the code related to langchain or Langgraph so the Glue code is not monitored in lanchain
# touching-sol

high-level overview transaction lifecycle, block production, slot progression, PoH, BFT etc

```mermaid
graph TD

  subgraph Time
    Epoch_Start[Epoch Starts] --> Leader_Schedule[Leader Schedule Determined Stake-based]
    Leader_Schedule --> Slot_Start_N{Slot N Starts ~400ms}
    Slot_Start_N --> Slot_End_N[Slot N Ends]
    Slot_End_N --> Slot_Start_N_plus_1{Slot N+1 Starts}
    Slot_Start_N_plus_1 --> Slot_End_N_plus_1[Slot N+1 Ends]
    Slot_End_N_plus_1 --> Slot_Start_N
  end


  Client[User/Client Application] --> Tx_Submit[Submit Transaction]
  Tx_Submit --> Tx_Pending[Tx Status: Pending]


  Tx_Pending --> Validator_Receive_Tx[Validator Node Receives Tx e.g., RPC]
  Validator_Receive_Tx --> Leader_Forward_Tx[Forward Tx to Current Leader]

  Slot_Start_N --> Leader_Assign_N[Validator N Assigned as Leader for Slot N]


  Leader_Assign_N --> Leader_Fetch_Tx[Leader N: Fetch Transactions]
  Leader_Forward_Tx --> Leader_Fetch_Tx
  Leader_Fetch_Tx --> Leader_Process_Tx[Leader N: Process Transactions SigVerify, Banking]


  Leader_Process_Tx --> Tx_Valid_Leader{Tx Valid?}
  Tx_Valid_Leader -- Yes --> Tx_Processed[Tx Status: Processed by Leader]
  Tx_Valid_Leader -- No --> Tx_Failed_Leader[Tx Status: Failed Invalid Signature/State]


  Leader_Process_Tx --> Leader_PoH[Leader N: Add Events/Tx to PoH Sequence]
  Leader_PoH_ContinuousPoH[Generator Running Continuously] --- Leader_PoH


  Tx_Processed & Leader_PoH --> Leader_Create_Block[Leader N: Create Block N Bundle Tx, PoH, Prev Hash]


  Leader_Create_Block --> Leader_Propagate_Block[Leader N: Propagate Block N to Network]


  Leader_Propagate_Block --> Validators_Receive_Block[Other Validators: Receive Block N]
  Validators_Receive_Block --> Validators_Validate_Block[Other Validators: Validate Block N Tx Integrity, PoH Check]


  Validators_Validate_Block --> Block_Valid{Block N Valid?}
  Block_Valid -- No --> Block_Invalid[Block N Invalid]
  Block_Invalid --> Validators_Ignore_Block[Validators Ignore Invalid Block]

  Block_Valid -- Yes --> Validators_Vote[Other Validators: Vote on Block N Stake-Weighted]


  Validators_Vote --> Network_Accumulate_Votes[Network: Accumulate Votes Tower BFT]
  Network_Accumulate_Votes --> Block_Confirmed{Supermajority Vote 2/3+ for Block N?}


  Block_Confirmed -- Yes --> Block_Finalized[Block N Finalized]
  Block_Confirmed -- No --> Network_Accumulate_Votes


  Leader_Create_Block --> Tx_Included_in_Block[Tx Included in Block N]
  Tx_Included_in_Block --> Validators_Validate_Block

  Block_Confirmed -- Yes --> Tx_Confirmed_Status[Tx Status: Confirmed]
  Block_Finalized --> Tx_Finalized[Tx Status: Finalized]


  Leader_Fetch_Tx --> Tx_Not_Included{Tx Not Fetched/Included by Leader?}
  Tx_Not_Included -- Yes --> Tx_Still_Pending[Tx Status: Still Pending]
  Tx_Still_Pending --> Validator_Receive_Tx

  Block_Invalid --> Tx_Failed_Block[Tx Status: Failed In Invalid Block]


  Leader_Assign_N --> Leader_Fail{Leader N Fails to Produce Block?}
  Leader_Fail -- Yes --> Slot_End_N


  Slot_Start_N_plus_1 --> Leader_Assign_N_plus_1[Validator N+1 Assigned as Leader for Slot N+1]
  Leader_Assign_N_plus_1 --> Leader_Fetch_Tx


  classDef time fill:#e0e0e0,stroke:#333;
  class Epoch_Start,Slot_Start_N,Slot_End_N,Slot_Start_N_plus_1,Slot_End_N_plus_1,Leader_Schedule time;

  classDef tx fill:#ffc,stroke:#333;
  class Tx_Submit,Tx_Pending,Tx_Processed,Tx_Included_in_Block,Tx_Confirmed_Status,Tx_Finalized,Tx_Failed_Leader,Tx_Failed_Block,Tx_Still_Pending tx;

  classDef validator fill:#f9f,stroke:#333,stroke-width:2px;
  class Leader_Assign_N,Leader_Fetch_Tx,Leader_Process_Tx,Leader_Create_Block,Leader_Propagate_Block,Validators_Receive_Block,Validators_Validate_Block,Validators_Vote,Leader_Assign_N_plus_1 validator;

  classDef poh fill:#ccf,stroke:#333,stroke-width:2px;
  class Leader_PoH,Leader_PoH_Continuous poh;

  classDef consensus fill:#afa,stroke:#333,stroke-width:2px;
  class Network_Accumulate_Votes,Block_Confirmed,Block_Finalized consensus;

  classDef decision fill:#ff9,stroke:#333,stroke-width:2px;
  class Tx_Valid_Leader,Block_Valid,Block_Confirmed,Tx_Not_Included,Leader_Fail decision;

  classDef failure fill:#f99,stroke:#333;
  class Tx_Failed_Leader,Tx_Failed_Block,Block_Invalid,Validators_Ignore_Block failure;

  classDef process fill:#ddd,stroke:#333;
  class Client,Validator_Receive_Tx,Leader_Forward_Tx process;

  classDef note fill:#eee,stroke:#999,stroke-dasharray: 5 5;
  class Leader_PoH_Continuous note;
```

1. **Time Subgraph:** Shows the fundamental structure of Epochs and Slots, and how the Leader Schedule is determined per epoch. It illustrates the continuous progression of slots.
2. **Transaction Origin:** A transaction starts with a user/client and enters a pending state.
3. **Propagation & Leader:** The transaction is sent to a validator (like an RPC node) and forwarded towards the current slot's leader. The diagram shows the leader being assigned based on the schedule for the current slot (N).
4. **Leader Pipeline:** The assigned leader fetches transactions from its pool and puts them through its processing pipeline (SigVerify, Banking).
5. **Tx Validation at Leader:** Transactions are validated. Valid ones proceed; invalid ones lead to a "Failed (Invalid)" status.
6. **Proof of History (PoH):** The leader continuously generates the PoH sequence, incorporating processed transactions and events.
7. **Block Creation:** Valid, processed transactions and the current state of the PoH sequence are bundled into Block N.
8. **Block Propagation:** The leader sends Block N to other validators.
9. **Validation by Others:** Other validators receive Block N and validate its contents, including the transactions and the PoH sequence.
10. **Block Validation Outcome:** If the block is invalid (e.g., contains invalid transactions the leader missed, or a PoH discrepancy), validators ignore it, and transactions within it lead to a "Failed (In Invalid Block)" status. If valid, validators proceed to vote.
11. **Consensus (Tower BFT):** Validators cast stake-weighted votes. The network accumulates these votes.
12. **Confirmation & Finality:** If Block N receives a supermajority (>2/3) of stake votes, it becomes finalized.
13. **Transaction Status Updates:**
    * A transaction included in a block transitions from "Processed" to "Included in Block".
    * If the block is confirmed, the transaction status becomes "Confirmed".
    * If the block is finalized, the transaction status becomes "Finalized".
14. **Alternative Cases:**
    * **Tx Not Included:** A transaction might be valid but not fetched or included by the leader (e.g., arrived too late, network congestion). It remains "Pending" and re-enters the propagation pool to potentially be included by a future leader.
    * **Leader Failure:** If the assigned leader fails, the slot might end without a block from that leader.
    * **Block Invalid:** As mentioned, leads to Tx failure status.
15. **Slot Progression:** The diagram explicitly shows the transition to Slot N+1 and the assignment of the next leader, highlighting that this happens continuously, often before previous blocks are finalized.

## Credits

* dhruvsol / [git](https://github.com/dhruvsol)
* Arpita / [git](https://github.com/ArpitaGanatra)

# Mini Chain Take Home Technical
- **Duration:** 1:15 - 2:00
- **Overview:** Answer technical and conceptual questions and patch a couple holes in this small blockchain.
- We fully expect you to use LLM support! We're more interested in what your answers reveal about your approach to programming.
- This technical comes in **FIVE PARTS**:
    - Some warm-up questions. Called out in code and in this document with [W#]
    - Programming tasks. Called out in code and in this document with [P#].
    - Simulation tasks. Called out in code and in this document with [S#].
        - These should be achieved by your completion of the programming tasks, but test e2e performance of the system.
    - Rust specific questions. Called out in code and in this document with [R#].
    - Conceptual questions. Called out in code and in this document with [C#].

## The Mini Chain
Mini Chain is a small proof of work blockchain. In its current state, it lacks both block verification and chain selection algorithms. Your programmatic task will be to add these in.

### Architecture 
The basic architecture of MiniChain is defined in three traits: `Proposer`, `Miner`, and `Verifier`.

#### `Proposer`
The proposer reads transactions from the mempool and proposes blocks.

```rust
#[async_trait]
pub trait Proposer {

    // Builds a block from the mempool
    async fn build_block(&self) -> Result<Block, String>;

    // Sends a block to the network for mining
    async fn send_proposed_block(&self, block : Block)->Result<(), String>;

    // Propose a block to the network and return the block proposed
    async fn propose_next_block(&self) -> Result<Block, String> {
        let block = self.build_block().await?;
        self.send_proposed_block(block.clone()).await?;
        Ok(block)
    }

}
```

#### `Miner`
The miner receives proposed blocks and submits them for verification.
```rust
#[async_trait]
pub trait Miner {

    // Receives a block over the network and attempts to mine
    async fn receive_proposed_block(&self) -> Result<Block, String>;

    // Sends a block to the network for validation
    async fn send_mined_block(&self, block : Block) -> Result<(), String>;

    // Mines a block and sends it to the network
    async fn mine_next_block(&self) -> Result<Block, String> {
        let block = self.receive_proposed_block().await?;
        // default implementation does no mining
        self.send_mined_block(block.clone()).await?;
        Ok(block)
    }

}
```

#### `Verifier`
The verifier receives mined blocks, verifiers them, and updates its representation of the chain.
```rust
#[async_trait]
pub trait Verifier {

    // Receives a block over the network and attempts to verify
    async fn receive_mined_block(&self) -> Result<Block, String>;

    // Verifies a block
    async fn verify_block(&self, block: &Block) -> Result<bool, String>;

    // Synchronizes the chain
    async fn synchronize_chain(&self, block: Block) -> Result<(), String>;

    // Verifies the next block
    async fn verify_next_block(&self) -> Result<(), String> {
        let block = self.receive_mined_block().await?;
        let valid = self.verify_block(&block).await?;
        if valid {
            self.synchronize_chain(block).await?;
        }
        Ok(())
    }

}
```

#### `FullNode`
A `FullNode` combines the `Proposer`, `Miner`, `Verifier`, and more to provide a complete utility for running the blockchain.
```rust
#[async_trait]
pub trait FullNode 
    : Initializer + 
    ProposerController + MinerController + VerifierController + 
    MempoolController + MempoolServiceController +
    BlockchainController + Synchronizer + SynchronizerSerivce {

    /// Runs the full node loop
    async fn run_full_node(&self) -> Result<(), String> {
        self.initialize().await?;
        loop {
            let error = try_join!(
                self.run_proposer(),
                self.run_miner(),
                self.run_verifier(),
                self.run_mempool(),
                self.run_blockchain(),
                self.run_mempool_service(),
                self.run_synchronizer_service()
            ).expect_err("One of the controllers stopped without error.");

            println!("Error: {}", error);
        }

    }

}
```

### Helpful Code
Generally speaking you'll find the following useful

#### `BlockchainOperations::pretty_print_tree(&self)`
This will show you a marked up version of the block tree. It annotates the main chain, the chain tip, leaves, block height, etc. It will produce output like the below.
```
Full Node #1
└── (0) Block: f1534392279bddbf 
    └── (1) Block: 001c9f15b438d405 
        ├── (2) Block: 00b53870fa564e32 < TIP
        ├── (2) Block: 002fcec250aa0e43
        ├── (2) Block: 000815e5bd5161bf
        └── (2) Block: 00bf6f30ed06136f
```

#### Tuning `fast_chain` default for `BlockchainMetadata`
`BlockchainMetadata::fast_chain` is supposed to contain a set of parameters for the chain that enable it to be run by several full nodes on the same device. Tuning these parameters will help you produce different outcomes and ideally pass the simulation tests.
```rust
impl BlockchainMetadata {
    pub fn fast_chain() -> Self {
        BlockchainMetadata {
            query_slots: 4,
            slot_secs: 2,
            fork_resolution_slots : 2,
            block_size: 128,
            maximum_proposer_time: 1,
            mempool_reentrancy_secs: 2,
            transaction_expiry_secs: 8,
            difficulty: 2,
            proposer_wait_ms : 50
        }
    }
}
```

#### The tests
Generally speaking the tests which are not simulations are quite useful. Most of them should not be affected by your work and serve more as indications of how various systems work. However, the tests in `mini_chain/chain.rs` will likely be very relevant to confirming the desired behavior.

### The Assessment
This technical comes in five parts as mentioned above.

#### The Warmup
A selection of short exercises to get your brain going and get familiarized with the source.

##### [W1]
This function to check whether the chain has been resolved (leaves pruned) is unimplemented. Fix that. 

**HINT:** Count the leaves!
```rust
async fn is_resolved(&self) -> bool {
    // Check if there's only a single leaf
    // TODO: [W1] this is unimplemented.
    false
}
```

##### [W2]
This implementation of a method to generate a random Argument could blow the stack. Fix it.
```rust
impl Argument {
    pub fn random<R: Rng>(rng: &mut R, depth: u64) -> Self {

        // TODO [W2]: This is wrong. Prevent this from blowing the stack.
        let partition = if depth > 0 { 9 } else { 8 }; 

        match rng.gen_range(0..partition) {
            0 => Argument::U8(rng.gen()),
            1 => Argument::U16(rng.gen()),
            2 => Argument::U32(rng.gen()),
            3 => Argument::U64(rng.gen()),
            4 => Argument::U128(rng.gen()),
            5 => Argument::Address(Address::random(rng)),
            6 => Argument::Signer(Signer(Address::random(rng))),
            7 => {
                let len = rng.gen_range(1..8);
                let args: Vec<Argument> = (0..len).map(|_| Argument::random(rng, depth - 1)).collect();
                Argument::Vector(args)
            },
            8 => {
                let len = rng.gen_range(1..8);
                let args: Vec<Argument> = (0..len).map(|_| Argument::random(rng, depth - 1)).collect();
                Argument::Struct(args)
            },
            _ => unreachable!(),
        }
    }
}
```

#### Programming
Programming tasks.

##### [P1]
We have not implemented verification for proof of work. Implement this using a leading-zeros alg. x
```rust
// TODO [P1]: Implement proof of work verification for a difficulty (leading zeros).
pub fn verify_proof_of_work(&self, difficulty: usize) -> bool {
    false
}
```

##### [P2]
Implement your favorite chain selection alg. Remember, we store a block tree as an adjacency matrix in a `HashSet`. x

**HINT:** Longest chain is likely the most reasonable alg to implement.
```rust
async fn get_main_chain(&self) -> Result<Vec<Block>, String> {

    // TODO [P2]: Implement your preffered chain selection algorithm here
    Ok(Vec::new())
    
}
```

#### Entering the Simulation
Make the simulations pass the benchmarks.
##### [S1]
Generate passing output from the full node simulation in `mini_chain/node.rs`. Provide a screenshot of your final chain and an intermediate chain below.

##### [S2]
Generate passing output from the network simulation in `lib.rs`. Provide a screenshot of your final chain and an intermediate chain below.
![Screenshot 2024-12-19 at 5 47 55 PM](https://github.com/user-attachments/assets/9383667a-2403-4cac-9491-4364ff5da7f5)
![Screenshot 2024-12-19 at 5 48 27 PM](https://github.com/user-attachments/assets/3dbf8978-e502-4a96-b2ef-d4ca1247ad9b)
![Screenshot 2024-12-19 at 5 48 42 PM](https://github.com/user-attachments/assets/8f72a2e2-152c-4394-a481-1d3706078257)

#### Rustacean Station
Answer these Rust-oriented questions. Answer in the space below each question.

##### [R1]
In this crate, we use `async_channel` which is a multiple-produce, multiple-consumer channel to share a reference to a channel between threads. Why would sharing a reference to a `mpsc::Receiver` potentially dangerous? 

**HINT:** How is a receiver consumed?

##### [R2]
Compare and contrast the approaches to conditional traits used in `mini_chain/node.rs` and `mini_chain/mempool.rs`.

##### [R3]
Identify the pattern that is forced by the definition of the Verifier trait--assuming the Verifier would in fact mutate state.
```rust
// TODO [R3]: Identify the pattern that is forced by this trait--assuming the Verifier would in fact mutate state.
#[async_trait]
pub trait Verifier {

    // Receives a block over the network and attempts to verify
    async fn receive_mined_block(&self) -> Result<Block, String>;

    // Verifies a block
    async fn verify_block(&self, block: &Block) -> Result<bool, String>;

    // Synchronizes the chain
    async fn synchronize_chain(&self, block: Block) -> Result<(), String>;

    // Verifies the next block
    async fn verify_next_block(&self) -> Result<(), String> {
        let block = self.receive_mined_block().await?;
        let valid = self.verify_block(&block).await?;
        if valid {
            self.synchronize_chain(block).await?;
        }
        Ok(())
    }

}
```

##### [R4]
Explain how the `broadcast_message` function works in our `Network` simulator. Reference the functionality of `Sender` and `Receiver`.

```rust
// TODO [R4]: Explain how this broadcast_message function works
pub async fn broadcast_message<T: Clone + Send + Debug + 'static>(
    &self,
    ingress_receiver: Arc<RwLock<Receiver<T>>>,
    egress_senders: &[Sender<T>],
) -> Result<(), String> {
    loop {

        let message = {
            let mut receiver = ingress_receiver.write().await;
            receiver.recv().await.unwrap()
        };

        let mut broadcasts = Vec::new();
        for sender in egress_senders {
            let cloned_message = message.clone();
            broadcasts.push(async {
                if !self.metadata.read().await.should_drop() {
                    self.metadata.read().await.introduce_latency().await;
                    sender.send(cloned_message).await.unwrap();
                }
            });
        }
        futures::future::join_all(broadcasts).await;
    }
}
```

#### Blockchainceptual
Conceptual questions about the blockchain. Answer in the space below each question.

##### [C1]
What effect should we expect changing the number of transactions in a block to have on our Blockchain? Would it help or hurt temporary forking?

##### [C2]
What would happen if we did not initialize the chain representations with a genesis block?

##### [C3]
One of our simulation benchmarks was the chains' edit score. What qualities of our network does this best measure (if any)? Is there a better way to measure something similar.
```rust
fn chains_edit_score(main_chains : Vec<Vec<String>>) -> f64 {

    let mut score = 0.0;
    let mut max_score = 0.0;

    for (i, chain_a) in main_chains.iter().enumerate() {
        let length_a = chain_a.len();
        for (j, chain_b) in main_chains.iter().enumerate() {
            let length_b = chain_b.len();
            if i != j {
                score += levenshtein_distance_vec(chain_a, chain_b) as f64;
                max_score += (length_a + length_b) as f64;
            }
        }
    }

    1.0 - (score/max_score)

}
```

##### [C4]
Explain why we request blocks that are referenced by an incoming block but that we don't have in our chain. Why don't we just ignore them?

##### [C5]
Our blockchain works with a notion of time slots. Why should we expect consensus about when these slots start and end?

**HINT**: Thomas Schelling.
# MiniBlockchain

# post-quantum-cryptography-blockchain-summary
Summary of the workflow for the paper, "Towards Building Post-Quantum Secure Ethereum". 

Research Objective: To determine whether it is feasible to transition Ethereum to FALCON (https://falcon-sign.info/), a quantum-computer resistant cryptography scheme, from the elliptic curve digital signature algorithm (ECDSA) cryptography scheme, which is not quantum-computer resistant.

How we answer this: In the context of blockchain, how much slower would it be to use FALCON vs. ECDSA? Also, how much more storage would it consume? If it doesn't take magnitudes more time and storage, then it would be feasible.

Approach:
1. We want to test this in the fastest way possible. While building out a version of Ethereum that completely replaces ECDSA with FALCON is possible, we want to find the simplest way to test using FALCON first without redesigning the entire blockchain.
    - We did this by isolating out the uses of ECDSA within the Ethereum execution client, and replacing those use cases with FALCON. Then we can benchmark the different execution speeds
    - The 3 use cases of ECDSA are for:
        1. Key Generation
        2. Signature Generation
        3. Signature Verification
2. To build this test out the fastest way possible, we looked for open source implementations of FALCON and Ethereum. There is a popular open-source Rust version of the Ethereum execution client we used, called rETH (https://reth.rs/). There is also an open-source Rust version of FALCON (https://docs.rs/falcon-rust/latest/falcon_rust/) as well as a python version (https://github.com/tprest/falcon.py)
    - We used rETH in order to isolate out the use cases of ECDSA to know what we had to replace with FALCON
    - We saw that we needed to implement a public-key recovery version of FALCON in order to make it compatible with rETH, as Ethereum uses a public-key recovery version of ECDSA in order to minimize storage requirements. The public-key recovery version of FALCON isn't available within either open-source FALCON implementations, so we coded that version.
        - First we implemented public-key recovery FALCON in python to make sure it worked properly. This was done on "[falcon.py/Example.ipynb](https://github.com/0xHLiu/falcon.py/blob/db104e25e617392ad60981dfdfb8ba817d55e935/Example.ipynb)" (see submodule), which is a fork of the python implementation of FALCON (https://github.com/tprest/falcon.py).
            - We ensured that it worked by running FALCON in a small test case (n=2) to check the calculations by hand
            - Then we ran it over 100 times to ensure that the implementation never failed
            - This was done mainly to the sign_pk_recovery(sk, message, randombytes=urandom) and verify_pk_recovery(sk, pk, message, signature) functions in "[falcon.py/Example.ipynb](https://github.com/0xHLiu/falcon.py/blob/db104e25e617392ad60981dfdfb8ba817d55e935/Example.ipynb)"
        - After ensuring that the python implementation was correct, we implemented it in Rust. We mainly worked on "[pq-ec-cryptography-benchmarks/falcon-rust/src/falcon.rs](https://github.com/0xHLiu/pq-ec-cryptography-benchmarks/blob/53f06c2f0517f14753ebde3d5ef9d7a1a190cf98/falcon-rust/src/falcon.rs)" (see submodule), which is a fork of the Rust implementation of FALCON (https://github.com/aszepieniec/falcon-rust).
            - You can find the altered portions by looking for "#[cfg(feature = "pk_recovery_mode")]", which means that when we turn on the configuration to use public-key recovery mode, that code is then used
            - Within the PublicKey Struct, we changed the function to generate the secret key
                - pub fn from_secret_key(sk: &SecretKey<N>) -> Self
            - Within the Signature Struct, we changed the serialization and de-serialization of the signature
            - We changed the signature generation function
                - pub fn sign<const N: usize>(m: &[u8], sk: &SecretKey<N>) -> Signature<N>
            - We changed the signature verification function
                - pub fn verify<const N: usize>(m: &[u8], sig: &Signature<N>, pk: &PublicKey<N>) -> bool
            - Lastly we built custom tests to ensure that it all worked correctly. We took examples from the python implementation that worked correctly and pasted them into Rust. You can find the tests for this by looking for "#[test]"
        - Now, the rust implementation of public-key recovery FALCON is working correctly. This would be the version of FALCON that would be plugged into rETH if the function runtimes isn't 10x slower than that of ECDSA and if the storage requirements isn't 10x larger
3. We wrote a benchmark test for our pk-recovery version of FALCON and for ECDSA to compare the function runtimes. This is done in "[pq-ec-cryptography-benchmarks/benchmark/benches](https://github.com/0xHLiu/pq-ec-cryptography-benchmarks/tree/53f06c2f0517f14753ebde3d5ef9d7a1a190cf98/benchmark/benches)" (see submodules).
    - This is done using the crate, "criterion" (https://bheisler.github.io/criterion.rs/book/analysis.html)
    - The benchmarking worked in 4 steps:
    1. Warmup: The functions are run a few times to warmup the cache
    2. Measurement: The functions are run multiple times in a sample (d) that will take about 100ms to execute in total. For example, if a function took 10ms, the function will be run 10 times to generate a sample (d). Next, the function is run 2d times, 3d times, and so on, generating a set of results of execution times for the function running [d, 2d, 3d ... Nd] times.
    3. Analysis: OLS is done on the collected samples (function execution time vs. # of function executions), and the slope output is the function runtime. This is done multiple times with new sets of bootstrapped data, sampled from the data collected before, to generate the mean, median, and 95% confidence interval for the function runtime.
    4. Comparison: The results of the function execution time get compared to the execution time measured before in previous benchmark runs if the benchmark has been run previously.
    - The results were that pk-recovery FALCON is 3x slower in signature verification than ECDSA, and that is the bottleneck function in public blockchains. Also, pk-recovery FALCON is 28x slower and 22,000x slower in signature generation and key generation compared to ECDSA, but those functions aren't run as often and aren't as impactful as signature verification speeds for a well functioning blockchain.
4. The storage requirement changes for pk-recovery FALCON was estimated by looking at the signature sizes, which would need to be stored on-chain, along with how much of the storage in blockchains are taken up by transaction signatures. From this, we determined that block sizes would increase about 4-fold, meaning that a blockchain would be 4-times larger.
5. We posted the results, data, and figures in "[PQ_Thesis_Results](https://github.com/0xHLiu/PQ_Thesis_Results/tree/f928bcda5d2a11f19a9742a4ff9dac65a2dd8ab7)" (see submodule).

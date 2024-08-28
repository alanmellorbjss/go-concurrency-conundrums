# Close channels

TODO

- no need to close
- signal no more values
- receive behaviour (default values)
- range provides "no more data" check
- Example of closed channel sending default values to receives, need to check via range
- No equivalent of channel.IsClosed() - make clear

# Data race

- What is it? Example code
- Data race detector
- Using mutexes
- Disadvantages in throughput at scale with mutexes

- channels preferred

- Rules to avoid race (no shared data otherthan channels)

- Example of a data race caused by the receive happening before the send go routine has time to start (unbuffered channel)

TODO

# Done channel

- Explain signalling purpose
- Demo of chan bool closed
- Go idiom chan struct{} - signalling only

# Waitgroups

- Provide good example
- Bad example: Data race example (Add called from wrong goroutine)

# Fan out: Worker pool

- Describe what they are
- Exercise: work queue, multiple receivers, waitgroup to end program

TODO

# Fan in - reducer

- Describe
- Exercise: work queue, fan out to process eg prepend text or assign a local-generated int sequence ID, fan-in to collate and sort in order

# Further - Actor model

- Brief description

# Further Reading

[Further reading >>](/further-reading.md)

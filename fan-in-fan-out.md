# Close channels

TODO

# Data race

TODO

# Done channel

# Waitgroups

- Good example
- Data race example (Add called from wrong goroutine)

# Misc - fold in where fits

- Example of a data race caused by the receive happening before the send go routine has time to start (unbuffered channel)
- Example of closed channel sending default values to receives, need to check via range
- No equivalent of channel.IsClosed() - make clear

# Fan out: Worker pool

- Describe what they are
- Exercise: work queue, multiple receivers, waitgroup to end program

TODO

# Fan in - reducer

- Describe
- Exercise: work queue, fan out to process eg prepend text or assign a local-generated int sequence ID, fan-in to collate and sort in order

# Further Reading

[Further reading >>](/further-reading.md)

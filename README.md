# Go Concurrency Conundrums

Goroutines and channels are the flagship features of the Go language. Providing an alternative to threads, these features simplify concurrent programming while making best use of available CPU resources.

_Simplified_ is not quite the same as _simple_, however.

The following exercises are a mix of ideas that work, and _conundrums_ - puzzles that don't work, for you to figure out why.

Before we start, make sure you're familiar with the course material on concurrency:

- [Concurrency 101](https://bjss.sharepoint.com/:p:/s/BJSSAcademy/EYTaDYKsSm5JoESht4apHOoBAvXxj--BnBBd2GJ0nmkYJg)
- [Concurrency student handout](https://bjss.sharepoint.com/:w:/s/BJSSAcademy/ERtqtYo7urxLrm2K8q9VMwEBOKr2dxDvgkf8gcuT5GtOUw)
- [Concurrency topics - links](https://bjss.sharepoint.com/:w:/s/BJSSAcademy/EaFGRmXbiT1KlXi1GBmk-OUBvocOko4bUwoni6Exis4UQQ)

Next, let's review the rules governing how the Go Scheduler runs concurrent programs:

[Goroutine rules overview >>](/goroutine-rules-overview.md)

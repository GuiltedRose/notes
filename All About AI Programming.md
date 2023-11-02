# Why Rust?

We are building in rust for the fact that it will cut compile times down, normally this isn't an issue, but only having access to a laptop, and the elimination of garbage collection are equally important to me, and I would like this to run smoothly with as little graphics card intervention as possible.

The other reasons are as follows:
* Most ML principles are basic algebraic expressions
* Rust is a lot easier to read than a bunch of python if/else nested branches (not always the case, but still a small issue for my eyes).
* I feel Rust actually can do more over time.

I might not be able to give detailed information of each step listed, as there aren't enough frameworks built for rust on this topic. There are however, some rust implementations of python packages we all love such as:
* NumPy
* Pandas
* pytorch

There are more, but these ate the ones I know off the top of my head. Most of these aren't even a full rewrite, but a quick port to get a job done. You really don't need these, as they will just speed up the process, and for someone like myself who loves to learn how things work I feel this is a perfect opportunity to do it myself with only allowing myself to use the random dependency for math.

# Why rewrite everything myself?

Short Answer: To prove that rust can be easier when done the "rust way".

So to do this setup, we need to do a few things first. For example: I am starting with a neural network. To accomplish this we should create it as a rust module so that our `main.rs` file doesn't get too huge too fast. This is actually the same way I'd rewrite my OS project (not posting until I figure some stuff out first).

# How to Create Rust Modules

You need to make a separate folder within the `src` directory. This should be the same name as the rs file in question, and contain a `[file name].rs` file. This is to input the module into our main file when we are ready to bring it all together. This is why a lot of complex code bases have a bunch of dependencies, but we can limit this ourselves by doing a lot of the heavy lifting ourselves.
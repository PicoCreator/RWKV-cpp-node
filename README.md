# RWKV.cpp NodeJS bindings

Arguably the easiest way to get RWKV.cpp running on node.js

> This is not a pure JS solution, and depends on the [precompiled RWKV.cpp binaries found here](https://github.com/saharNooby/rwkv.cpp)

> This currently runs purely on your CPU, while that means you can use nearly anything to run it, you also do not get any insane speed up with a GPU (yet)

# What is RWKV?

RWKV, is a LLM which which can switch between "transformer" and "RNN" mode.

This gives it the best of both worlds
- High scalable training in transformer
- Low overheads when infering each token in RNN mode

Along with the following benefits
- Theoretically Infinite context size
- Embedding support via hidden states

For more details on the math involved, and how this model works on a more technical basis. [Refer to the official project](https://github.com/BlinkDL/RWKV-LM)


# Setup 

Install the node module

```.bash
npm i rwkv-cpp-node
```

Download one of the prequantized rwkv.cpp weights, from hugging face (raven, is RWKV pretrained weights with fine-tuned instruction sets)

- [RWKV raven 7B v11 Q8_0](https://huggingface.co/BlinkDL/rwkv-4-raven/resolve/main/Q8_0-RWKV-4-Raven-7B-v11x-Eng99%25-Other1%25-20230429-ctx8192.bin)
- [RWKV raven 7B v11 Q8_0 (multilingual, performs worse in english)](https://huggingface.co/BlinkDL/rwkv-4-raven/resolve/main/Q8_0-RWKV-4-Raven-7B-v11-Eng49%25-Chn49%25-Jpn1%25-Other1%25-20230430-ctx8192.bin)
- [RWKV raven 14B v11 Q8_0](https://huggingface.co/BlinkDL/rwkv-4-raven/resolve/main/Q8_0-RWKV-4-Raven-14B-v11x-Eng99%25-Other1%25-20230501-ctx8192.bin)

Alternatively you can download one of the [raven pretrained weights from the hugging face repo](https://huggingface.co/BlinkDL/rwkv-4-raven/tree/main). 
And perform your own quantization conversion using the [original rwkv.cpp project](https://github.com/saharNooby/rwkv.cpp)

# Usage

```.javascript
const RWKV = require("RWKV-cpp-node");

// Load the module with the pre-qunatized cpp weights
const raven = new RWKV("<path-to-your-model-bin-files>")

// Call the completion API
let res = raven.completion("RWKV is a")

// And log, or do something with the result
console.log( res.completion )
```

Advance setup options

```.javascript
// You can setup with the following parameters with a config object (instead of a string path)
const raven = new RWKV({
	// Path to your cpp weights
	path: "<path-to-your-model-bin-files>",

	// Threads count to use, this is auto detected based on your number of vCPU
	// if its not configured
	threads: 8,

	//
	// Cache size for the RKWV state, This help optimize the repeated RWKV calls
	// in use cases such as "conversation", allow it to skip the previous chat computation
	//
	// it is worth noting that the 7B model takes up about 2.64 MB for the state buffer, 
	// meaning you will need atleast 264 MB of RAM for a cachesize of 100
	//
	// This defaults to 50
	// Set to false or 0 to disable
	//
	stateCacheSize: 50
});
```

Completion API options

```.javascript
// Lets perform a completion, with more options
let res = raven.completion({

	// The prompt to use
	prompt: "<prompt str>",

	// Completion default settings
	// See openai docs for more details on what these do for your output if you do not understand them
	// https://platform.openai.com/docs/api-reference/completions
	max_tokens: 64,
	temperature: 1.0,
	top_p: 1.0,
	stop: [ "\n" ],

	// Streaming of output, either token by token, or the full complete output stream
	streamCallback: function(tokenStr, fullCompletionStr) {
		// ....
	},

	// Existing RWKV hidden state, represented as a Flaot32Array
	// do not use this unless you REALLY KNOW WHAT YOUR DOING
	//
	// This will skip the state caching logic 
	initState: (Special Float32Array)
});

// Additionally if you have a commonly reused instruction set prefix, you can preload this
// using either of the following (requires the stateCacheSize to not be disabled)
raven.preloadPrompt( "<prompt prefix string>" )
raven.completion({ prompt:"<prompt prefix string>", max_tokens:0 })
```

Completion output format

```.javascript
// The following is a sample of the result object format
let resFormat = {
	// Completion generated
	completion: '<completion string used>',
	completionTokens: [ <int values representing the completion tokens> ],

	// Prompt used
	prompt: '<prompt string used>',
	promptTokens: [ <int values representing the prompt tokens> ],

	// Token usage numbers
	usage: {
		promptTokens: 41,
		completionTokens: 64,
		totalTokens: 105,
		// number of tokens in the prompt that was previously cached
		promptTokensCached: 39 
	},

	// Performance statistics of the completion operation
	//
	// the following perf numbers is from a single 
	// `Intel(R) Xeon(R) CPU E5-2695 v3 @ 2.30GHz`
	// an old 2014 processor, with 28 vCPU
	// 
	perf: {
		// Time taken in ms for each segment
		promptTime: 954,
		completionTime: 35907,
		totalTime: 36861,

		// Time taken in ms to process each token at the respective phase
		timePerPrompt: 477, // This excludes cached tokens
		timePerCompletion: 561.046875,
		timePerFullPrompt: 23.26829268292683, // This includes cached tokens (if any)

		// The average tokens per second
		promptPerSecond: 2.0964360587002098, // This excludes cached tokens
		completionPerSecond: 1.7823822652964603,
		fullPromptPerSecond: 42.9769392033543 // This includes cached tokens (if any)
	}
}
```

# What can be improved?

- [Add GPU support via RWKV-cpp-cuda project](https://github.com/harrisonvanderbyl/rwkv-cpp-cuda)
- [RWKV-tokenizer-node library performance](https://github.com/PicoCreator/RWKV-tokenizer-node/issues/1)
- [Add MMAP support for RWKV.cpp](https://github.com/saharNooby/rwkv.cpp/issues/50)
- [Reducing JS and RWKV.cpp back and forth for prompt eval](https://github.com/saharNooby/rwkv.cpp/pull/49)
- Validate and add support for X arch / OS 
	- If your system is not supported, try to do a build on the rwkv.cpp project
		- [Add it to the lib folder](https://github.com/PicoCreator/RWKV-cpp-node/tree/main/lib)
		- [Modify the OS / Architecture detection code](https://github.com/PicoCreator/RWKV-cpp-node/blob/main/src/cpp_bind.js#L19)
- Utility function to download the model weights / quantize them ??
- CLI tooling to quickly setup / download ??
- varient of `preloadPrompt` which gurantee the saved prompt does not get cache evicted ??

# How to run the unit test?

```.bash
npm run test
```

# Clarification notes

**Why didn't you cache the entire prompt?**

I intentionally did not cache the last 2 tokens, to avoid sub-optimal performance when the prompt strings, should have been merged as a single token, which would have impacted the quality of result.

For example "Hello" represents a single token of 12092

However if I cached every prompt blindly in full, if you performed multiple calls, character by character. Each subsequent call would continue from the previous cached result in its "incomplet form".

As a result when you finally call "Hello", it can end up consisting of 5 tokens, with 1 character each. (aka ["H","e","l","l","o"])
This leads to extreamly unexpected behaviour in the quality of the model output.

While the example is an extreame case, there are smaller scale off-by-1 example regarding whitespace.

# Special thanks & refrences

@saharNooby - original rwkv.cpp implementation

- https://github.com/saharNooby/rwkv.cpp

@BlinkDL - for the main rwkv project

- hhttps://github.com/BlinkDL/RWKV-LM

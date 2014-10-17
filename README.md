Synapse
=======

Synapse.js is a javascript/node.js neural network library, its generalized algorythm is architecture-free, so you can build and train basically any type of first order or even [second order neural network](http://en.wikipedia.org/wiki/Recurrent_neural_network#Second_Order_Recurrent_Neural_Network) architectures.

This library includes a few built-in architectures like [multilayer perceptrons](http://en.wikipedia.org/wiki/Multilayer_perceptron), [multilayer long-short term memory](http://en.wikipedia.org/wiki/Long_short_term_memory) networks (LSTM) or [liquid state machines](http://en.wikipedia.org/wiki/Liquid_state_machine), and a trainer capable of training any given network, which includes built-in training tasks/tests like solving an XOR, completing a Distracted Sequence Recall task or an [Embeded Reber Grammar](http://www.willamette.edu/~gorr/classes/cs449/reber.html) test, so you can easily test and compare the performance of different architectures.


The algorythm implemented by this library has been taken from Derek D. Monner's paper:

[A generalized LSTM-like training algorithm for second-order recurrent neural networks](http://www.overcomplete.net/papers/nn2012.pdf)


There are references to the equations in that paper commented through the source code.

##Overview

###Installation

######In node
You can install synapse with [npm](http://npmjs.org):

`npm install synapse --save`

######In the browser
Just include the file synapse.js (you can find it in the `/lib` directory) with a script tag in your HTML:

`<script src="synapse.js"></script>`

###Usage

```
var synapse = require('synapse');
var Neuron = synapse.Neuron,
	Layer = synapse.Layer,
	Network = synapse.Network,
	Trainer = synapse.Trainer,
	Architect = synapse.Architect;
```

Now you can start to create networks, train them, or use built-in networks from the [Architect](http://github.com/cazala/synapse#architect).

###Examples

######Perceptron

This is how you can create a simple perceptron.

```
function Perceptron(input, hidden, output)
{
	// create the layers
	var inputLayer = new Layer(input);
	var hiddenLayer = new Layer(hidden);
	var outputLayer = new Layer(output);

	// connect the layers
	inputLayer.project(hiddenLayer);
	hiddenLayer.project(outputLayer);

	// set the layers
	this.set({
		input: inputLayer,
		hidden: [hiddenLayer],
		output: outputLayer
	});
}

// extend the prototype chain
Perceptron.prototype = new Network();
Perceptron.prototype.constructor = Perceptron;
```

Now you can test your new network by creating a trainer and teaching the perceptron to learn an XOR

```
var myPerceptron = new Perceptron(2,3,1);
var myTrainer = new Trainer(myPerceptron);

myTrainer.XOR(); // { error: 0.004998819355993572, iterations: 21871, time: 356 }

myPerceptron.activate([0,0]); // 0.0268581547421616
myPerceptron.activate([1,0]); // 0.9829673642853368
myPerceptron.activate([0,1]); // 0.9831714267395621
myPerceptron.activate([1,1]); // 0.02128894618097928
```

######Long Short-Term Memory

This is how you can create a simple long short-term memory with input gate, forget gate, output gate, and peephole connections.

```
function LSTM(input, blocks, output)
{
	// create the layers
	var inputLayer = new Layer(input);
	var inputGate = new Layer(blocks);
	var forgetGate = new Layer(blocks);
	var memoryCell = new Layer(blocks);
	var outputGate = new Layer(blocks);
	var outputLayer = new Layer(output);

	// connections from input layer
	var input = inputLayer.project(memoryCell);
	inputLayer.project(inputGate);
	inputLayer.project(forgetGate);
	inputLayer.project(outputGate);

	// connections from memory cell
	var output = memoryCell.project(outputLayer);

	// self-connection
	var self = memoryCell.project(memoryCell, Layer.connectionType.ONE_TO_ONE);

	// peepholes
	memoryCell.project(inputGate,  Layer.connectionType.ONE_TO_ONE);
	memoryCell.project(forgetGate, Layer.connectionType.ONE_TO_ONE);
	memoryCell.project(outputGate, Layer.connectionType.ONE_TO_ONE);

	// gates
	inputGate.gate(input, Layer.gateType.INPUT);
	forgetGate.gate(self, Layer.gateType.ONE_TO_ONE);
	outputGate.gate(output, Layer.gateType.OUTPUT);

	// input to output direct connection
	inputLayer.project(outputLayer);

	// set the layers of the neural network
	this.set({
		input: inputLayer,
		hidden: hiddenLayers,
		output: outputLayer
	});
}

// extend the prototype chain
LSTM.prototype = new Network();
LSTM.prototype.constructor = LSTM;
```

These are examples for explanatory purposes, the [Architect](http://github.com/cazala/synapse#architect) already includes Multilayer Perceptrons and
Multilayer LSTM networks architectures.


Documentation
=============

##Neuron

Neurons are the basic unit of the neural network. They can be connected together, or used to gate connetions between other neurons. 
A Neuron can perform basically 4 operations: project connections, gate connections, activate and propagate.

######project

A neuron can project a connection to another neuron (i.e. connect neuron A with neuron B).
Here is how it's done:

```
var A = new Neuron();
var B = new Neuron();
A.project(B); // A now projects a connection to B
```

Neurons can also self-connect:

`A.project(A);`

The method **project** returns a `Connection` object, that can be gated by another neuron.

######gate

A neuron can gate a connection between two neurons, or a neuron's self-connection. This allows you to create [second order neural network](http://en.wikipedia.org/wiki/Recurrent_neural_network#Second_Order_Recurrent_Neural_Network) architectures.

```
var A = new Neuron();
var B = new Neuron();
var connection = A.project(B);

var C = new Neuron();
C.gate(connection); // now C gates the connection between A and B
```


######activate

When a neuron activates, it computes it's state from all its input connections and squashes it using its activation function, and returns the output (activation).
You can provide the activation as a parameter (useful for neurons in the input layer. it has to be a float between 0 and 1). For example:

```
var A = new Neuron();
var B = new Neuron();
A.project(B);

A.activate(0.5); // 0.5
B.activate(); // 0.3244554645
```


######propagate

After an activation, you can teach the neuron what should have been the output (a.k.a. train). This is done by backpropagating the error.
To use the **propagate** method you have to provide a learning rate, and a target value (float between 0 and 1).

For example, if I want to train neuron B to output a value close to 0 when neuron A activates a value of 1, with an error smaller than 0.005:

```
var A = new Neuron();
var B = new Neuron();
A.project(B);

var learningRate = .3,
	targetOutput = 0,
	error = 0.005,
	output = 1;

while(output - targetOutput > error)
{
	A.activate(1);

	output = B.activate();
	B.propagate(learningRate, targetOutput);
}

// test it
A.activate(1);
B.activate(); // 0.0049998578298219975
```

######squashing function and bias

By default, a neuron uses a [Logistic Sigmoid](http://en.wikipedia.org/wiki/Logistic_function) as its squashing/activation function, and a random bias. 
You can change those properties the following way:

```
var A = new Neuron();
A.squash = Neuron.squash.HTAN;
A.bias = 1;
```

There are 4 built-in squashing functions, but you can also create your own:

- Neuron.squash.LOGISTIC
- Neuron.squash.HTAN
- Neuron.squash.IDENTITY
- Neuron.squash.HLIM

##Layer

Normally you won't work with single neurons, but use Layers instead. A layer is basically an array of neurons, they can do pretty much the same things as neurons do, put it makes the programming process faster.

To create a layer you just have to specify its size (the number of neurons in that layer).

`var myLayer = new Layer(5)`

######project

A layer can project a connection to another layer. You have to provide the layer that you want to connect to and the `connectionType`:

```
var A = new Layer(5);
var B = new Neuron(3);
A.project(B, Layer.connectionType.ALL_TO_ALL); // All the neurons in layer A now project a connection to all the neurons in layer B
```

Layers can also self-connect:

`A.project(A, Layer.connectionType.ONE_TO_ONE);`

There are two `connectionType`'s:
- Layer.connectionType.ALL_TO_ALL
- Layer.connectionType.ONE_TO_ONE

If you make a one-to-one connection, both layers must have **the same size**.

The method **project** returns a `LayerConnection` object, that can be gated by another layer.

######gate

A layer can gate a connection between two other layers, or a layers's self-connection.

```
var A = new Neuron();
var B = new Neuron();
var connection = A.project(B);

var C = new Neuron();
C.gate(connection, Layer.gateType.INPUT_GATE); // now C gates the connection between A and B (input gate)
```

There are three `gateType`'s:
- Layer.gateType.INPUT_GATE: If layer C is gating connections between layer A and B, all the neurons from C gate all the input connections to B.
- Layer.gateType.OUTPUT_GATE: If layer C is gating connections between layer A and B, all the neurons from C gate all the output connections from A.
- Layer.gateType.ONE_TO_ONE: If layer C is gating connections between layer A and B, each neuron from C gates one connection from A to B. This is useful for gating self-connected layers. To use this kind of gateType, A, B and C must be the same size.

######activate

When a layer activates, it just activates all the neurons in that layer in order, and returns an array with the outputs. It accepts an array of activations as parameter (for input layers):

```
var A = new Layer(5);
var B = new Layer(3);
A.project(B);

A.activate([1,0,1,0,1]); // [1,0,1,0,1]
B.activate(); // [0.3280457, 0.83243247, 0.5320423]
```


######propagate

After an activation, you can teach the layer what should have been the output (a.k.a. train). This is done by backpropagating the error.
To use the **propagate** method you have to provide a learning rate, and a target value (array of floats between 0 and 1).

For example, if I want to train layer B to output [0,0] when layer A activates [1,0,1,0,1], with an error smaller than 0.005:

```
var A = new Layer(5);
var B = new Layer(2);
A.project(B, Layer.connectionType.ALL_TO_ALL);

var learningRate = .3,
	targetOutput = [0,0],
	error = 0.005
	output = [1,1];

while(output[0] - targetOutput[0] > error && output[1] - targetOutput[1] > error)
{
	A.activate([1,0,1,0,1]);

	output = B.activate();
	B.propagate(learningRate, targetOutput);
}

// test it
A.activate([1,0,1,0,1]);
B.activate(); // [0.004999850993267468, 0.00499980138183861]
```

######squashing function and bias

You can set the squashing function and bias of all the neurons in a layer by using the method **set**:

```
myLayer.set({
	squash: Neuron.squash.HTAN,
	bias: 0
})
```





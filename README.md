# <a href="https://mjmcguffin.github.io/muqcs.js/">muqcs.js</a>

Mucqs (pronounced mucks) is McGuffin's Useless Quantum Circuit Simulator
(named in an allusion to mush, Moser's Useless Shell).  It is written in JavaScript, and allows one to simulate circuits programmatically or from a command line.  It has no graphical front end, does not leverage the GPU for computations, and does not import any special libraries.

The code is contained entirely in a single file, and defines a small class for complex numbers, a class for complex matrices (i.e., matrices storing complex numbers), and a few utility classes.  These classes take up less than a thousand lines of code.  The rest of the code consists of a regression test (in the function performRegressionTest()) followed by some performance tests.  Having a relatively small amount of source code means that the code can be more easily understood by others.

To run the code, load the html file into a browser like Chrome, and then open a console (in Chrome, this is done by selecting 'Developer Tools').  From the console prompt, you can call functions in the code and see output printed to the console.

**Creating and Manipulating Matrices**

To create some matrices and print out their contents, we can do

    let m0 = new CMatrix(2,3);
    console.log("A 2x3 matrix filled with zeros:\n" + m0.toString());
    let m1 = CMatrix.create([[10,20],[30,40]]);
    console.log("A 2x2 matrix:\n" + m1.toString());
    let c0 = CMatrix.createColVector([5,10,15]);
    console.log("The transpose of a column vector is a row vector:\n" + c0.transpose().toString());

which produces this output:

    A 2x3 matrix filled with zeros:
    [_,_,_]
    [_,_,_]
    A 2x2 matrix:
    [10,20]
    [30,40]
    The transpose of a column vector is a row vector:
    [5,10,15]

Notice that the toString() method returns a string containing newline characters.  In the source code, this is referred to as a 'multiline string'.
Next, we create a matrix containing complex numbers, add two matrices together, and print out the matrices and their sum:

    let m2 = CMatrix.create([[new Complex(0,1),new Complex(2,3)],[new Complex(5,7),new Complex(-1,-3)]]);
    let m3 = CMatrix.sum(m1,m2);
    console.log(StringUtil.concatenateMultilineStrings(m1.toString()," + ",m2.toString()," = ",m3.toString()));

which produces this output:

    [1,2] + [1i  ,2+3i ] = [1+1i,4+3i]
    [3,4]   [5+7i,-1-3i]   [8+7i,3-3i]

Similarly, there are static methods in the CMatrix class for subtracting matrices (diff(m1,m2)), multiplying matrices (mult(m1,m2) and naryMult([m1,m2,...])), and for computing their tensor product (tensor(m1,m2) and naryTensor([m1,m2,...])).
The CMatrix class also has some predefined vectors and matrices.  For example,

    console.log(CMatrix.ketOne.toString());

prints the column vector |1>:

    [_]
    [1]

and

    console.log(CMatrix.gate4x4cnot.toString());

prints the 4x4 matrix for the CNOT gate:

    [1,_,_,_]
    [_,_,_,1]
    [_,_,1,_]
    [_,1,_,_]

Notice that zeros are replaced with underscores to make it easier for a human to read sparse matrices (to change this behavior, you can call toString({suppressZeros:false}) or toString({charToReplaceSuppressedZero:'.'})).  You might also notice that the matrix for the CNOT gate looks different from the way it is usually presented in textbooks or other sources.  This is related to the ordering of bits and ordering of tensor products.  Search for the usingTextbookConvention flag in the source code for comments that explain this, and set that flag to true if you prefer the textbook ordering.  We can also call a method on a matrix (or a vector) to change its ordering:

    console.log(CMatrix.gate4x4cnot.reverseEndianness().toString());

prints the 4x4 matrix for the CNOT gate in its more usual form:

    [1,_,_,_]
    [_,1,_,_]
    [_,_,_,1]
    [_,_,1,_]

**Simulating a Quantum Circuit**

To simulate a circuit, there are two approaches.  The first involves storing one explicit matrix for each stage of the circuit.  In a circuit with N qubits, the matrices will have size 2^N x 2^N.  Here we see an example of how to simulate a 3-qubit circuit with this first approach:

    // Simulate a circuit on three qubits
    // equivalent to
    //     https://algassert.com/quirk#circuit={%22cols%22:[[%22X^%C2%BC%22,%22Y^%C2%BC%22,%22H%22],[1,%22X^%C2%BC%22],[1,%22X%22,%22%E2%80%A2%22]]}
    //
    // qubit q0 |0>----(x^0.25)-------------------------
    //
    // qubit q1 |0>----(y^0.25)-----(x^0.25)----(+)-----
    //                                           |
    // qubit q2 |0>-------H----------------------o------
    //
    input = CMatrix.naryTensor( [ CMatrix.ketZero /*q2*/, CMatrix.ketZero /*q1*/, CMatrix.ketZero /*q0*/ ] );
    step1 = CMatrix.naryTensor( [ CMatrix.gate2x2hadamard /*q2*/, CMatrix.gate2x2fourthrooty /*q1*/, CMatrix.gate2x2fourthrootx /*q0*/ ] );
    step2 = CMatrix.naryTensor( [ CMatrix.gate2x2identity /*q2*/, CMatrix.gate2x2fourthrootx /*q1*/, CMatrix.gate2x2identity /*q0*/ ] );
    step3 = CMatrix.expand4x4ForNWires( CMatrix.gate4x4cnot, 2, 1, 3 );
    output = CMatrix.naryMult([ step3, step2, step1, input ]);
    console.log(StringUtil.concatenateMultilineStrings(
        step3.toString(),
        " * ",
        step2.toString({decimalPrecision:1}),
        " * ",
        "...", // step1.toString(),
        " * ",
        input.toString(),
        " = ",
        output.toString(true)
    ));

Each matrix takes up O((2^N)^2) space, and calling CMatrix.mult() on two such matrices would cost O((2^N)^3) time.
However, the call to naryMult() above causes the matrices to be multiplied right-to-left, because naryMult() checks the sizes of the matrices to optimize the multiplication order, and the right-most matrix passed to naryMult() is just a column vector of size 2^N x 1.  The matrix just before has size 2^N x 2^N, and multiplying the two together costs O((2^N)^2) and produces another column vector, which gets multiplied by the next matrix before them. 
Hence, in this first approach, the space and time requirements of each step of the circuit are O((2^N)^2).
Notice in the last call to toString() above, we pass in a true value; this causes bit strings like |000> to be printed in front of the matrix, as a reminder of the association between base states and matrix rows.
The output is:

    [1,_,_,_,_,_,_,_]   [0.9+0.4i,0       ,0.1-0.4i,0       ,0       ,0       ,0       ,0       ]         [1]   [0.302+0.479i]
    [_,1,_,_,_,_,_,_]   [0       ,0.9+0.4i,0       ,0.1-0.4i,0       ,0       ,0       ,0       ]         [_]   [0.198-0.125i]
    [_,_,1,_,_,_,_,_]   [0.1-0.4i,0       ,0.9+0.4i,0       ,0       ,0       ,0       ,0       ]         [_]   [0.302+0.125i]
    [_,_,_,1,_,_,_,_] * [0       ,0.1-0.4i,0       ,0.9+0.4i,0       ,0       ,0       ,0       ] * ... * [_] = [0.052-0.125i]
    [_,_,_,_,_,_,1,_]   [0       ,0       ,0       ,0       ,0.9+0.4i,0       ,0.1-0.4i,0       ]         [_]   [0.302+0.125i]
    [_,_,_,_,_,_,_,1]   [0       ,0       ,0       ,0       ,0       ,0.9+0.4i,0       ,0.1-0.4i]         [_]   [0.052-0.125i]
    [_,_,_,_,1,_,_,_]   [0       ,0       ,0       ,0       ,0.1-0.4i,0       ,0.9+0.4i,0       ]         [_]   [0.302+0.479i]
    [_,_,_,_,_,1,_,_]   [0       ,0       ,0       ,0       ,0       ,0.1-0.4i,0       ,0.9+0.4i]         [_]   [0.198-0.125i]

A second approach to simulating the same circuit is to not store any explicit matrices of size 2^N x 2^N.  Instead, we only store the state vector of size 2^N x 1, and update it for each stage of the circuit.  The following code does this:

    input = CMatrix.naryTensor( [ CMatrix.ketZero /*q2*/, CMatrix.ketZero /*q1*/, CMatrix.ketZero /*q0*/ ] );
    step1 = CMatrix.transformStateVectorWith2x2(CMatrix.gate2x2hadamard,2,3,input,[]);
    step1 = CMatrix.transformStateVectorWith2x2(CMatrix.gate2x2fourthrooty,1,3,step1,[]);
    step1 = CMatrix.transformStateVectorWith2x2(CMatrix.gate2x2fourthrootx,0,3,step1,[]);
    step2 = CMatrix.transformStateVectorWith2x2(CMatrix.gate2x2fourthrootx,1,3,step1,[]);
    output = CMatrix.transformStateVectorWith2x2(CMatrix.gate2x2not,1,3,step2,[[2,true]]);
    console.log(StringUtil.concatenateMultilineStrings(
        input.toString(),
        " -> ",
        step1.toString(),
        " -> ",
        step2.toString(),
        " -> ",
        output.toString(true)
    ));

In this second approach, the space and time requirements of each step of the circuit are O(2^N), so, much better than in the first approach.
The magic happens in the CMatrix.transformStateVectorWith2x2() method, which is based on Quirk’s source code https://github.com/Strilanc/Quirk/ , in particular, Quirk's applyToStateVectorAtQubitWithControls() method in src/math/Matrix.js 

**Conventions**

In a circuit with N qubits, the wires are numbered 0 for the top wire to (N-1) for the bottom wire.  The top wire encodes the Least-Significant Bit (LSB).

**Limitations**

The code can generate a swap matrix via the CMatrix.wireSwap() method, but this is only usable with the first approach for simulation outlined above.  With the second simulation approach, there is no support for swap gates.  And there is no support at all for controlled swap gates, in either approach.


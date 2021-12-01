---
layout: post
title:  "Learning Verilator: Sorting Hardware"
date:   2021-11-30 17:24:00 -0000
categories: fpga
---

Recently I took MIT's digital systems laboratory class (6.111) and I learned
how to design hardware in Verilog. Before I took the class, I read about
Verilator, a SystemVerilog simulator. At the time, I didn't know how to use
Verilator and I didn't know how to write Verilog. Learning both at the same
time was too hard and I gave up. Now that I know Verilog, I want to come back
and try Verilator again.

For those of you new to Verilator too, it's a tool for simulating hardware
described in SystemVerilog. It does that by compiling the SystemVerilog source
into C++ or SystemC code. We can then build that C++ code and run the
simulation. You can learn more about it in the
[Verilator website](https://www.veripool.org/verilator/).

Since I mostly want to learn about Verilator, I stuck to a simple RTL project:
a module that sorts 8 32-bit unsigned integers and outputs them sequentially
in sorted order. In this post, I'll walk you through the steps I went from
start to finish. Since I'm still learning, this post is more of a journal/blog
than a guide, you can definitely find better guides out there, but it could be
useful to someone. I also had a lot of fun with the project, so maybe someone
will find it fun too. Let's get to it!

I like to start a project with organizing where all the bits are going to go.
For this project, I'm collecting all my design files under rtl/ and my test
benches under bench/.

My design process always starts with the interface. I knew I wanted to try
implementing a bitonic network, so I started with that.
```systemverilog
module bitonic4(input wire clk, 
                input wire[31:0] din[0:WIDTH-1],
                output logic[31:0] dout[0:WIDTH-1]);
```

Then, the test bench.
```cpp
#include <iostream>
#include <memory>
#include <verilated.h>
#include <verilated_vcd_c.h>
#include "Vbitonic.h"

const int MAX_SIM_TIME = 20;
vluint64_t sim_time = 0;

int main(int argc, char* argv[]) {
    std::unique_ptr<Vbitonic> dut = std::make_unique<Vbitonic>();

  Verilated::traceEverOn(true);
  VerilatedVcdC *m_trace = new VerilatedVcdC;
  dut->trace(m_trace, 5);
  m_trace->open("waveform.vcd");

  for (int i = 0; i < 4; i++) {
    dut->din[i] = i;
  }

  for (sim_time = 0; sim_time < MAX_SIM_TIME; sim_time++) {
    dut->clk ^= 1;
    dut->eval();
    m_trace->dump(sim_time);
  }

  m_trace->close();
  return 0;
}
```

I'm not going to get into the implementation details of the bitonic network
here, but you can find it in the [repo](https://github.com/gilbertmike/sorting-machine).
The implementation is just Verilog translation of a diagram from [Wikipedia](https://en.wikipedia.org/wiki/Bitonic_sorter).

Now comes our first interaction with Verilator. The first step is to generate
the C++ files. We call Verilator:
```
verilator -Wall --trace --cc rtl/bitonic.sv --exe bench/bitonic_tb.cpp --top-module bitonic4
```
The `--trace` means we will be generating traces to view with GTKWave later.
The `--top-module` specifies the top level module in our design. The `-Wall`
flag enables all warnings. The `--cc` flag says that we want C++ generated
instead of SystemC.

This will create a directory obj_dir/ which contains the generated files.
Verilator will also generate a Makefile Vbitonic.mk in obj_dir/. We can now
build the simulation binary.
```
make -C obj_dir -f Vbitonic.mk Vbitonic
```
A binary obj_dir/Vbitonic is generated. We can run this binary to get a trace
we can view with GTKWave.

Since we will be calling this often (I didn't get my code right the first time,
of course), I put the two commands in a Makefile.
```
bitonic:
    verilator -Wall --trace --cc rtl/bitonic.sv --exe bench/bitonic_tb.cpp --top-module bitonic4
    ${MAKE} -C obj_dir -f Vbitonic.mk Vbitonic
```

And that's it! 

The design has another module `sorter` that uses `bitonic4`. The steps are
basically the same.

And finally, the waveform we've been waiting for.
![Waveform of sorter](https://github.com/gilbertmike/sorting-machine/blob/main/sorter.png)

That's it from me. Thanks for reading!

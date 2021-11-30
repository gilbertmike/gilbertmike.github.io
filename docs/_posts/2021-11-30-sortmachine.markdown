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

---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

An automated unit testing tool for Dafny. 

[**Purpose**](#purpose) <br>
[**How to Install?**](#how-to-install-artemis) <br>
[**Getting Started**](#getting-started) <br>
[**Overview of Command Line Options**](#overview-of-command-line-options) <br>

## Purpose

Artemis is a tool that generate runtime tests for Dafny code. The tests could be used:
- to ensure a compiled program preserves the behavior verified in Dafny;
- to increase confidence in specifications of external libraries;
- to increase assurance that a Dafny program is functionally equivalent to an existing implementation that may be written in another language.

## How to Install Artemis?

We have tested Artemis on Ubuntu and Mac. To install the tool:
1. Clone the modified version of Dafny: `git clone --recursive https://github.com/Dargones/dafny.git -b LatestPlus`
2. Install `z3` version `4.8.5` as described in the official Dafny guide ([for Linux](https://github.com/dafny-lang/dafny/wiki/INSTALL#linux-source)) ([for Mac](https://github.com/dafny-lang/dafny/wiki/INSTALL#Mac-binary))
3. Build the cloned repository with dotnet: `dotnet build Source/Dafny.sln`

## Getting Started

Consider the following simple Dafny program stored in a file called `Program.dfy` (a slightly modified version of [this method](https://github.com/aws/aws-encryption-sdk-dafny/blob/fd2516f9d919ccff05ccd14f9ff158c11fb42fa1/src/Util/Sorting.dfy#L50-L57) from AWS Encryption SDK):

```dafny
module Module {
  newtype uint8 = x | 0 <= x < 0x100
  function method LexLeq(x: seq<uint8>, y: seq<uint8>, n: nat):(result:bool)
    requires n <= |x| && n <= |y| 
    ensures result ==> n == |x| || n != |y| 
    decreases |x| - n 
  {
      n == |x| 
  || (n != |y| && x[n] < y[n]) 
  || (n != |y| && x[n] == y[n] && LexLeq(x, y, n + 1))
  }
}
```

To generate tests for this program, run the following command (see below for a breakdown of the command-line options):

```bash
dotnet <YOUR_DAFNY_FOLDER>/Binaries/Dafny.dll /generateTestMode:Block /timeLimit:5 /generateTestTargetMethod:Module.LexLeq /generateTestOracle:Spec /generateTestSeqLengthLimit:3 Program.dfy
```

You should get the following output:

```dafny
include "Program.dfy"
module ProgramdfyUnitTests {
  import Module
  method {:test} test0() {
    var d0 : seq<Module.uint8> := [(0 as Module.uint8), (133 as Module.uint8), (188 as Module.uint8)];
    var d1 : seq<Module.uint8> := [(0 as Module.uint8), (133 as Module.uint8), (187 as Module.uint8)];
    expect (1 as nat) <= |d0| && (1 as nat) <= |d1|, "Test does not meet preconditions and should be removed";
    var r0 := Module.LexLeq(d0, d1, (1 as nat));
    expect r0 ==> (1 as nat) == |d0| || (1 as nat) != |d1|;
  }
  // More tests go here...
  method {:test} test4() {
    var d0 : seq<Module.uint8> := [(0 as Module.uint8)];
    var d1 : seq<Module.uint8> := [(0 as Module.uint8), (0 as Module.uint8)];
    expect (1 as nat) <= |d0| && (1 as nat) <= |d1|, "Test does not meet preconditions and should be removed";
    var r0 := Module.LexLeq(d0, d1, (1 as nat));
    expect r0 ==> (1 as nat) == |d0| || (1 as nat) != |d1|;
  }
}
```

You can optionally specify a `/generateTestPrintBpl:<FILENAME.bpl>` command line option, which should print the Boogie program that Artemis uses to generate tests (it is slightly different from the standard Dafny to Boogie translation).

To compile the tests to C\#, assuming you have put the tests above in a file called `ProgramUnitTests.dfy` and it is in the same folder as `Program.dfy`, run:

```bash
dotnet <YOUR_DAFNY_FOLDER>/Binaries/Dafny.dll /spillTargetCode:3 /compile:0 /noVerify ProgramUnitTests.dfy
```

In particular, `test0` from above will be translated to:

```cs
public static void test0() {
  Dafny.ISequence<byte> _0_d0;
  _0_d0 = Dafny.Sequence<byte>.FromElements((byte)(0), (byte)(133), (byte)(188));
  Dafny.ISequence<byte> _1_d1;
  _1_d1 = Dafny.Sequence<byte>.FromElements((byte)(0), (byte)(133), (byte)(187));
  if (!(((BigInteger.One) <= (new BigInteger((_0_d0).Count))) && ((BigInteger.One) <= (new BigInteger((_1_d1).Count))))) {
    throw new Dafny.HaltException("/home/sasha/Desktop/artfact/aws-encryption-sdk-dafny/ProgramUnitTests.dfy(7,0): " + Dafny.Sequence<char>.FromString("Test does not meet preconditions and should be removed"));
  }
  bool _2_r0;
  _2_r0 = Module_Compile.__default.LexLeq(_0_d0, _1_d1, BigInteger.One);
  if (!(!(_2_r0) || (((BigInteger.One) == (new BigInteger((_0_d0).Count))) || ((BigInteger.One) != (new BigInteger((_1_d1).Count)))))) {
    throw new Dafny.HaltException("/home/sasha/Desktop/artfact/aws-encryption-sdk-dafny/ProgramUnitTests.dfy(9,0): " + Dafny.Sequence<char>.FromString("expectation violation"));
  }
}
```

Finally, to execute the tests, first create a `Program.csproj` file with the following code in it:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net6.0</TargetFramework>
    </PropertyGroup>
    <ItemGroup>
        <PackageReference Include="Microsoft.NET.Test.Sdk" Version="16.10.0" />
        <PackageReference Include="xunit" Version="2.4.1" />
        <PackageReference Include="xunit.runner.visualstudio" Version="2.4.3" />
        <PackageReference Include="Moq" Version="4.16.1" />
    </ItemGroup>
</Project>
```

Once done, run `dotnet test Program.csproj` to execute the tests.

## Overview of Command Line Options

Artemis supports the following command line options:



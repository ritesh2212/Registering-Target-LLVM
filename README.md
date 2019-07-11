# Registering-A-New-Target-LLVM

In this repo we are registering a New backend to LLVM 8.x. This repo contains bare minimum files to register a target to LLVM which can be viewed by 
```
llc --version
```
## Getting Started

These instructions will get you to register a new target so follow these as it is.

### llvm_src/CMakeLists.txt
```
set(LLVM_ALL_TARGETS
  ...
  Cpu0
  ...
  )
```

### llvm_src/include/llvm/ADT/Triple.h
```
...
#undef mips
#undef cpu0
...
class Triple {
public:
  enum ArchType {
    ...
    cpu0,       // For Tutorial Backend Cpu0
    cpu0el,
    ...
  };
  ...
}
```
### llvm_src/include/llvm/Object/ELFObjectFile.h
```
...
template <class ELFT>
StringRef ELFObjectFile<ELFT>::getFileFormatName() const {
  switch (EF.getHeader()->e_ident[ELF::EI_CLASS]) {
  case ELF::ELFCLASS32:
    switch (EF.getHeader()->e_machine) {
    ...
    case ELF::EM_CPU0:        // llvm-objdump -t -r
      return "ELF32-cpu0";
    ...
  }
  ...
}
...
template <class ELFT>
unsigned ELFObjectFile<ELFT>::getArch() const {
  bool IsLittleEndian = ELFT::TargetEndianness == support::little;
  switch (EF.getHeader()->e_machine) {
  ...
  case ELF::EM_CPU0:  // llvm-objdump -t -r
    switch (EF.getHeader()->e_ident[ELF::EI_CLASS]) {
    case ELF::ELFCLASS32:
    return IsLittleEndian ? Triple::cpu0el : Triple::cpu0;
    default:
      report_fatal_error("Invalid ELFCLASS!");
    }
  ...
  }
}
```
### llvm_src/include/llvm/BinaryFormat/ELF.h
```
enum {
  ...
  EM_CPU0          = 999  // Document LLVM Backend Tutorial Cpu0
};
...
// Cpu0 Specific e_flags
enum {
  EF_CPU0_NOREORDER = 0x00000001, // Don't reorder instructions
  EF_CPU0_PIC       = 0x00000002, // Position independent code
  EF_CPU0_ARCH_32   = 0x50000000, // CPU032 instruction set per linux not elf.h
  EF_CPU0_ARCH      = 0xf0000000  // Mask for applying EF_CPU0_ARCH_ variant
};

// ELF Relocation types for Mips
enum {
#include "ELFRelocs/Cpu0.def"
};
...
```
### llvm_src/lib/Support/Triple.cpp
```
const char *Triple::getArchTypeName(ArchType Kind) {
  switch (Kind) {
  ...
  case cpu0:        return "cpu0";
  case cpu0el:      return "cpu0el";
  ...
  }
}
...
const char *Triple::getArchTypePrefix(ArchType Kind) {
  switch (Kind) {
  ...
  case cpu0:
  case cpu0el:      return "cpu0";
  ...
}
...
Triple::ArchType Triple::getArchTypeForLLVMName(StringRef Name) {
  return StringSwitch<Triple::ArchType>(Name)
    ...
    .Case("cpu0", cpu0)
    .Case("cpu0el", cpu0el)
    ...
}
...
static Triple::ArchType parseArch(StringRef ArchName) {
  return StringSwitch<Triple::ArchType>(ArchName)
    ...
    .Cases("cpu0",  Triple::cpu0)
    .Cases("cpu0el", Triple::cpu0el)
    ...
}
...
static Triple::ObjectFormatType getDefaultFormat(const Triple &T) {
  ...
  case Triple::cpu0:
  case Triple::cpu0el:
  ...
}
...
static unsigned getArchPointerBitWidth(llvm::Triple::ArchType Arch) {
  switch (Arch) {
  ...
  case llvm::Triple::cpu0:
  case llvm::Triple::cpu0el:
  ...
    return 32;
  }
}
...
Triple Triple::get32BitArchVariant() const {
  Triple T(*this);
  switch (getArch()) {
  ...
  case Triple::cpu0:
  case Triple::cpu0el:
  ...
    // Already 32-bit.
    break;
  }
  return T;
}
```

## Create the Initial Cpu0.td
### llvm_src/lib/Target/Cpu0/Cpu0.td
```
//===-- Cpu0.td - Describe the Cpu0 Target Machine ----*- tablegen -*-===//
//
//                     The LLVM Compiler Infrastructure
//
// This file is distributed under the University of Illinois Open Source
// License. See LICENSE.TXT for details.
//
//===----------------------------------------------------------------------===//
// This is the top level entry point for the Cpu0 target.
//===----------------------------------------------------------------------===//

//===----------------------------------------------------------------------===//
// Target-independent interfaces
//===----------------------------------------------------------------------===//

include "llvm/Target/Target.td"

```
### llvm_src/lib/Target/Cpu0/CmakeLists.txt
```
set(LLVM_TARGET_DEFINITIONS Cpu0.td)


add_llvm_target(Cpu0CodeGen
         Cpu0TargetMachine.cpp
         )
add_subdirectory(MCTargetDesc)
add_subdirectory(TargetInfo)

```
### llvm_src/lib/Target/Cpu0/LLVMBuild.txt
```
;===- ./lib/Target/Cpu0/LLVMBuild.txt --------------------------*- Conf -*--===;
;
;                     The LLVM Compiler Infrastructure
;
; This file is distributed under the University of Illinois Open Source
; License. See LICENSE.TXT for details.
;
;===------------------------------------------------------------------------===;
;
; This is an LLVMBuild description file for the components in this subdirectory.
;
; For more information on the LLVMBuild system, please see:
;
;   http://llvm.org/docs/LLVMBuild.html
;
;===------------------------------------------------------------------------===;

# Following comments extracted from http://llvm.org/docs/LLVMBuild.html

[common]
subdirectories = 
  MCTargetDesc TargetInfo

[component_0]
# TargetGroup components are an extension of LibraryGroups, specifically for 
#  defining LLVM targets (which are handled specially in a few places).
type = TargetGroup
# The name of the component should always be the name of the target. (should 
#  match "def Cpu0 : Target" in Cpu0.td)
name = Cpu0
# Cpu0 component is located in directory Target/
parent = Target
# Whether this target defines an assembly parser, assembly printer, disassembler
#  , and supports JIT compilation. They are optional.

[component_1]
# component_1 is a Library type and name is Cpu0CodeGen. After build it will 
#  in lib/libLLVMCpu0CodeGen.a of your build command directory.
type = Library
name = Cpu0CodeGen
# Cpu0CodeGen component(Library) is located in directory Cpu0/
parent = Cpu0
# If given, a list of the names of Library or LibraryGroup components which 
#  must also be linked in whenever this library is used. That is, the link time 
#  dependencies for this component. When tools are built, the build system will 
#  include the transitive closure of all required_libraries for the components 
#  the tool needs.
required_libraries =
                     CodeGen Core MC 
                     Cpu0Desc 
                     Cpu0Info 
                     SelectionDAG 
                     Support 
                     Target
# end of required_libraries

# All LLVMBuild.txt in Target/Cpu0 and subdirectory use 'add_to_library_groups 
#  = Cpu0'
add_to_library_groups = Cpu0
```
### llvm_src/lib/Target/Cpu0/Cpu0.h
```
//===-- Cpu0.h - Top-level interface for Cpu0 representation ----*- C++ -*-===//
//
//                     The LLVM Compiler Infrastructure
//
// This file is distributed under the University of Illinois Open Source
// License. See LICENSE.TXT for details.
//
//===----------------------------------------------------------------------===//
//
// This file contains the entry points for global functions defined in
// the LLVM Cpu0 back-end.
//
//===----------------------------------------------------------------------===//

#ifndef LLVM_LIB_TARGET_CPU0_CPU0_H
#define LLVM_LIB_TARGET_CPU0_CPU0_H

#include "MCTargetDesc/Cpu0MCTargetDesc.h"
#include "llvm/Target/TargetMachine.h"

namespace llvm {
  class Cpu0TargetMachine;
  class FunctionPass;
  Target &getTheCpu0Target();
} // end namespace llvm;

#endif
```
### llvm_src/lib/Target/Cpu0/Cpu0TargetMachine.cpp
```
//===-- Cpu0TargetMachine.cpp - Define TargetMachine for Cpu0 -------------===//
//
//                     The LLVM Compiler Infrastructure
//
// This file is distributed under the University of Illinois Open Source
// License. See LICENSE.TXT for details.
//
//===----------------------------------------------------------------------===//
//
// Implements the info about Cpu0 target spec.
//
//===----------------------------------------------------------------------===//

#include "Cpu0TargetMachine.h"
#include "Cpu0.h"

#include "llvm/IR/LegacyPassManager.h"
#include "llvm/CodeGen/Passes.h"
#include "llvm/CodeGen/TargetPassConfig.h"
#include "llvm/Support/TargetRegistry.h"
using namespace llvm;

#define DEBUG_TYPE "cpu0"

extern "C" void LLVMInitializeCpu0Target() {
//RegisterTargetMachine<Cpu0TargetMachine> registered_target(getTheCpu0Target());  
 }

```
### llvm_src/lib/Target/Cpu0/Cpu0TargetMachine.h
```
#ifndef LLVM_LIB_TARGET_CPU0_CPU0TARGETMACHINE_H
#define LLVM_LIB_TARGET_CPU0_CPU0TARGETMACHINE_H

#include "llvm/CodeGen/Passes.h"
#include "llvm/CodeGen/SelectionDAGISel.h"
#include "llvm/CodeGen/TargetFrameLowering.h"
#include "llvm/Target/TargetMachine.h"

#endif
```
### llvm_src/lib/Target/Cpu0/MCTargetDesc/CMakeLists.txt
```
add_llvm_library(LLVMCpu0Desc
  Cpu0MCTargetDesc.cpp
  )

```
### llvm_src/lib/Target/Cpu0/MCTargetDesc/Cpu0MCTargetDesc.cpp
```
//===-- Cpu0MCTargetDesc.cpp - Cpu0 Target Descriptions -------------------===//
//
//                     The LLVM Compiler Infrastructure
//
// This file is distributed under the University of Illinois Open Source
// License. See LICENSE.TXT for details.
//
//===----------------------------------------------------------------------===//
//
// This file provides Cpu0 specific target descriptions.
//
//===----------------------------------------------------------------------===//

#include "Cpu0MCTargetDesc.h"
#include "llvm/MC/MachineLocation.h"
#include "llvm/MC/MCELFStreamer.h"
#include "llvm/MC/MCInstrAnalysis.h"
#include "llvm/MC/MCInstPrinter.h"
#include "llvm/MC/MCInstrInfo.h"
#include "llvm/MC/MCRegisterInfo.h"
#include "llvm/MC/MCSubtargetInfo.h"
#include "llvm/MC/MCSymbol.h"
#include "llvm/Support/CommandLine.h"
#include "llvm/Support/ErrorHandling.h"
#include "llvm/Support/FormattedStream.h"
#include "llvm/Support/TargetRegistry.h"

using namespace llvm;
extern "C" void LLVMInitializeCpu0TargetMC() {

}

```
### llvm_src/lib/Target/Cpu0/MCTargetDesc/Cpu0MCTargetDesc.h
```
//===-- Cpu0MCTargetDesc.h - Cpu0 Target Descriptions -----------*- C++ -*-===//
//
//                     The LLVM Compiler Infrastructure
//
// This file is distributed under the University of Illinois Open Source
// License. See LICENSE.TXT for details.
//
//===----------------------------------------------------------------------===//
//
// This file provides Cpu0 specific target descriptions.
//
//===----------------------------------------------------------------------===//

#ifndef LLVM_LIB_TARGET_CPU0_MCTARGETDESC_CPU0MCTARGETDESC_H
#define LLVM_LIB_TARGET_CPU0_MCTARGETDESC_CPU0MCTARGETDESC_H

#include "llvm/Support/DataTypes.h"

namespace llvm {
class Target;
class Triple;

extern Target TheCpu0Target;
extern Target TheCpu0elTarget;

} // End llvm namespace

#endif
```
### llvm_src/lib/Target/Cpu0/MCTargetDesc/LLVMBuild.txt
```
;===- ./lib/Target/Cpu0/MCTargetDesc/LLVMBuild.txt -------------*- Conf -*--===;
;
;                     The LLVM Compiler Infrastructure
;
; This file is distributed under the University of Illinois Open Source
; License. See LICENSE.TXT for details.
;
;===------------------------------------------------------------------------===;
;
; This is an LLVMBuild description file for the components in this subdirectory.
;
; For more information on the LLVMBuild system, please see:
;
;   http://llvm.org/docs/LLVMBuild.html
;
;===------------------------------------------------------------------------===;

[component_0]
type = Library
name = Cpu0Desc
parent = Cpu0
required_libraries = MC 
                     Cpu0Info 
                     Support
add_to_library_groups = Cpu0
```
### llvm_src/lib/Target/Cpu0/TargetInfo/CMakeLists.txt
```
add_llvm_library(LLVMCpu0Info
  Cpu0TargetInfo.cpp
  )
```
### llvm_src/lib/Target/Cpu0/TargetInfo/Cpu0TargetInfo.cpp
```
//===-- Cpu0TargetInfo.cpp - Cpu0 Target Implementation -------------------===//
//
//                     The LLVM Compiler Infrastructure
//
// This file is distributed under the University of Illinois Open Source
// License. See LICENSE.TXT for details.
//
//===----------------------------------------------------------------------===//

#include "Cpu0.h"
#include "llvm/IR/Module.h"
#include "llvm/Support/TargetRegistry.h"
using namespace llvm;
Target llvm::TheCpu0Target;//llvm::TheCpu0elTarget;

extern "C" void LLVMInitializeCpu0TargetInfo() {
  RegisterTarget<Triple::cpu0,true> X(TheCpu0Target, "cpu0", "Cpu0","Cpu0");

  
}
```
### llvm_src/lib/Target/Cpu0/TargetInfo/LLVMBuild.txt
```
;===- ./lib/Target/Cpu0/TargetInfo/LLVMBuild.txt ---------------*- Conf -*--===;
;
;                     The LLVM Compiler Infrastructure
;
; This file is distributed under the University of Illinois Open Source
; License. See LICENSE.TXT for details.
;
;===------------------------------------------------------------------------===;
;
; This is an LLVMBuild description file for the components in this subdirectory.
;
; For more information on the LLVMBuild system, please see:
;
;   http://llvm.org/docs/LLVMBuild.html
;
;===------------------------------------------------------------------------===;

[component_0]
type = Library
name = Cpu0Info
parent = Cpu0
required_libraries = Support
add_to_library_groups = Cpu0
```

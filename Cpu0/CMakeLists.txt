set(LLVM_TARGET_DEFINITIONS Cpu0.td)


add_llvm_target(Cpu0CodeGen
         Cpu0TargetMachine.cpp
         )
add_subdirectory(MCTargetDesc)
add_subdirectory(TargetInfo)


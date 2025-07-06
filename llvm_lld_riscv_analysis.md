# LLVM/LLD 中 RISC-V 汇编文件处理流程分析

## 概述

本报告分析LLVM/LLD如何处理RISC-V的.s（汇编文件）和.o（目标文件），特别关注data段信息的读取、解析和存储过程。

## 文件处理流程分层架构

### 1. LLVM汇编器处理 (.s 文件)

RISC-V .s文件的处理在LLVM的MC层（Machine Code层）完成：

#### 主要组件：
- **MCTargetAsmParser** - RISC-V特定的汇编解析器
- **MCStreamer** - 处理汇编指令流
- **MCObjectWriter** - 生成ELF目标文件

#### 处理位置：
```
llvm/lib/Target/RISCV/AsmParser/RISCVAsmParser.cpp
llvm/lib/Target/RISCV/MCTargetDesc/RISCVMCTargetDesc.cpp
llvm/lib/MC/MCObjectFileInfo.cpp
```

#### Data段处理时机：
1. **词法分析阶段**：识别`.data`、`.section .data`等指令
2. **语法分析阶段**：解析数据定义（`.word`, `.byte`, `.ascii`等）
3. **符号表构建**：记录data段中的符号（变量名、标签）
4. **重定位信息生成**：为data段中的符号引用生成重定位记录

### 2. LLD链接器处理 (.o 文件)

LLD处理.o文件的流程在ELF后端完成：

#### 核心数据结构：

**InputSectionBase** (`lld/ELF/InputSection.h:95-200`)：
```cpp
class InputSectionBase : public SectionBase {
  InputFile *file;                    // 所属文件
  uint32_t relSecIdx;                 // 重定位段索引
  const uint8_t *content_;            // 段内容
  uint64_t size;                      // 段大小
  SmallVector<Relocation, 0> relocations; // 重定位记录
  // ...
};
```

**ObjFile** (`lld/ELF/InputFiles.h:245-350`)：
```cpp
template <class ELFT> class ObjFile : public ELFFileBase {
  SmallVector<InputSectionBase *, 0> sections; // 所有段
  Symbol *symbols[];                   // 符号表
  ArrayRef<Elf_Word> shndxTable;      // 扩展段索引表
  // ...
};
```

## Data段信息读取的关键时间点

### 1. 文件解析阶段 (Driver.cpp:2700-2720)

```cpp
// lld/ELF/Driver.cpp:2715
for (size_t i = 0; i < files.size(); ++i) {
  parseFile(files[i]);  // 解析每个输入文件
}
```

### 2. 段初始化阶段 (InputFiles.cpp:730-800)

```cpp
// lld/ELF/InputFiles.cpp:730
template <class ELFT>
void ObjFile<ELFT>::initializeSections(bool ignoreComdats,
                                       const llvm::object::ELFFile<ELFT> &obj) {
  ArrayRef<Elf_Shdr> objSections = getELFShdrs<ELFT>();
  StringRef shstrtab = CHECK(obj.getSectionStringTable(objSections), this);
  
  for (size_t i = 0; i != size; ++i) {
    const Elf_Shdr &sec = objSections[i];
    // 处理每个段，包括data段
    switch (sec.sh_type) {
      // ...
      default:
        this->sections[i] = createInputSection(i, sec, 
          check(obj.getSectionName(sec, shstrtab)));
    }
  }
}
```

### 3. 符号初始化阶段 (InputFiles.cpp:1050-1120)

```cpp
// lld/ELF/InputFiles.cpp:1050
template <class ELFT>
void ObjFile<ELFT>::initializeSymbols(const object::ELFFile<ELFT> &obj) {
  ArrayRef<Elf_Sym> eSyms = this->getELFSyms<ELFT>();
  
  for (size_t i = firstGlobal, end = eSyms.size(); i != end; ++i) {
    const Elf_Sym &eSym = eSyms[i];
    // 解析符号信息：类型、大小、所属段等
    uint8_t type = eSym.getType();      // STT_OBJECT表示变量
    uint64_t value = eSym.st_value;     // 符号值（段内偏移）
    uint64_t size = eSym.st_size;       // 符号大小
    // ...
  }
}
```

### 4. 重定位扫描阶段 (Driver.cpp:2850-2870)

```cpp
// lld/ELF/Driver.cpp:2850
parallelForEach(ctx.objectFiles, [](ELFFileBase *file) {
  initSectionsAndLocalSyms(file, /*ignoreComdats=*/false);
});
```

## Data段变量信息的完整化过程

### 阶段1：基础信息读取 (文件解析时)

**时机**：`ObjFile::initializeSections()`  
**内容**：
- 段基本属性（flags, type, size, alignment）
- 段原始内容（字节数据）
- 段名称（".data", ".data.rel", etc.）

### 阶段2：符号信息关联 (符号解析时)

**时机**：`ObjFile::initializeSymbols()`  
**内容**：
- 变量名到段偏移的映射
- 变量大小信息
- 符号类型（STT_OBJECT用于变量）
- 符号绑定（STB_GLOBAL, STB_LOCAL等）

### 阶段3：重定位信息建立 (重定位处理时)

**时机**：`InputSection::relocate()`调用之前  
**内容**：
- 每个变量被引用的位置
- 重定位类型（R_RISCV_64, R_RISCV_32等）
- 引用来源（哪个函数/段引用了该变量）

### 阶段4：段间关系确定 (链接脚本处理时)

**时机**：`LinkerScript::processSectionCommands()`  
**内容**：
- Data段在输出文件中的最终位置
- 段之间的合并关系
- 对齐要求的最终确定

## Data段变量的数据结构表示

### 段级别信息：
```cpp
InputSectionBase {
  content_: const uint8_t*     // data段的原始字节内容
  size: uint64_t               // 段大小
  flags: uint64_t              // 段标志（SHF_WRITE | SHF_ALLOC）
  relocations: Vector<Relocation> // 重定位记录列表
}
```

### 变量级别信息：
```cpp
Defined {
  name: StringRef              // 变量名
  value: uint64_t              // 段内偏移
  size: uint64_t               // 变量大小
  section: InputSectionBase*   // 所属段
  type: uint8_t                // STT_OBJECT
}
```

### 重定位级别信息：
```cpp
Relocation {
  type: RelType                // 重定位类型
  offset: uint64_t            // 重定位位置（段内偏移）
  addend: int64_t             // 重定位加数
  sym: Symbol*                // 目标符号
}
```

## 实现Data段变量排序的关键位置

基于以上分析，实现data段变量按重定位次数/大小排序的最佳位置是：

### 推荐位置1：重定位扫描后 (Driver.cpp:2860-2870)
```cpp
// 在 initSectionsAndLocalSyms 之后
// 此时所有重定位信息已完整
parallelForEach(ctx.objectFiles, postParseObjectFile);
// 在这里添加data段变量排序逻辑
```

### 推荐位置2：段聚合阶段 (Driver.cpp:2920-2950)
```cpp
// lld/ELF/Driver.cpp:2920
llvm::TimeTraceScope timeScope("Aggregate sections");
for (InputFile *f : ctx.objectFiles) {
  for (InputSectionBase *s : f->getSections()) {
    // 在这里可以重新组织data段的内容
    if (s->name.starts_with(".data")) {
      // 执行变量排序和重组
    }
  }
}
```

## 关键发现总结

1. **重定位统计完成时机**：在`initSectionsAndLocalSyms()`调用后，所有重定位信息才完整可用

2. **段内容修改最佳时机**：在段聚合阶段（Driver.cpp:2920），此时可以创建新的合并段

3. **数据结构完整性**：`InputSectionBase`包含了实现排序所需的所有信息：
   - 段内容 (`content_`)
   - 重定位列表 (`relocations`)
   - 符号信息（通过`file->symbols`访问）

4. **架构限制确认**：LLD确实不支持单个InputSection内部的变量重排，必须通过创建新的合并段来实现全局排序

这个分析为实现data段变量按重定位密度排序提供了完整的技术路径和实现位置指导。
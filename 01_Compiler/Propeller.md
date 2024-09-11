# Propeller

从ThinLTO获取思想

```c++
bool Peephole::doDynamicGPRelaxing() {
  /**
   * 想做到动态GP，需要满足以下条件：
   * ① √对.data, .sdata的访问是基本按序的
   * ② gp的跳转范围远不够实现relax访问data段等
   * ③ 修改gp后要保证后续不会对原gp进行访问(即必须对jal等深度优先搜索)
   * ④ √使用lui+addi的次数足够多，使得总指令数得以减小
   * ⑤ √必要时需要对data段排序，当使用动态gp时无需考虑
   * lui	a2,0x39
   * addi	a2,a2,1735
   * jal	12602 <bm1publicfunc_846931022>
   * lui	a2,0x39
   * addi	a2,a2,1748 # 396d4 <__global_pointer$+0xfbe>
   * jal	1243e <bm1publicfunc_846930977>
   * lui	a2,0x39
   * ...
   * "强连通分量"（Strongly Connected Component, SCC）
   */
  // 第一步，识别主要发生在main函数，查找main函数范围
  uint32_t funcStart, funcEnd;
  for (const auto &symbol : symtable) 
    if (symbol.name == "main"){
      funcStart = symbol.value;
      funcEnd = funcStart + symbol.size;
      std::cout << "main: 0x" << std::hex << funcStart << " → 0x" << funcEnd << std::endl;
      break;
    }

  Imm __GP_Origin_;
  unsigned theorTotalBytes = 0;
  for (unsigned i = 0; i + 1 < insns.size(); i++){
    /** 第一步：确定初始gp值  */ 
    RV32Inst &I_1 = insns[i + 0];
    RV32Inst &I_2 = insns[i + 1];
    if (I_1.op == "AUIPC" && IH::isADDI(I_2) && I_1.rd == 3) {
      const Imm Hi20 = IH::getImm(I_1);
      const Imm Low12 = IH::getImm(I_2);
      __GP_Origin_ = (Hi20 << 12) + Low12 + I_1.offset;
      std::cout << "原GP = 0x" << std::hex << __GP_Origin_ << std::endl;
    }
    // 界定执行动态GP优化的范围
    if (insns[i].offset < funcStart || insns[i].offset >= funcEnd)
      continue;
    /** 第二步：找到需要relax的点  */ 
    if (IH::isLUI(I_1) && IH::isADDI(I_2)) {
      const Imm Hi20 = IH::getImm(I_1);
      const Imm Low12 = IH::getImm(I_2);
      unsigned jumpPoint = (Hi20 << 12) + Low12;
      // 跳转点在data, sdata中间的都可以和GP搭上关系
      if (jumpPoint < setable[3].start ||
        jumpPoint > setable[4].start + setable[4].size)
        continue;
      if (jumpPoint < __GP_Origin_ + 2048) // NOTE 临时限定！为了不影响GP前面无法跳转的
        continue;
      // std::cout << "当前所在指令为: 0x" << insns[i].offset << std::endl; 
    } else continue;
    /** 第三步：验证出现顺序是否为lui,addi,jal循环往复(界定子范围) */
    // 记录是否采纳lui,addi,jal组合
    bool accept = true;
    unsigned j = i;
    // 边界条件：只有循环超过四次才被采纳, 否则没有收益【当前指令为:I_1】
    int beginIdx = i, endIdx; // TODO 记录索引
    // 记录lui的高位
    const auto refHi = IH::getImm(I_1);
    // 记录满足的次数
    unsigned times = 0;
    while (j < insns.size()){
      // TODO JAL替换成SW也有可能正确
      if (IH::getImm(insns[j]) != refHi || insns[j].op != "LUI"){
        // 直到遇到了不满足的lui
        if (times > 4){
          endIdx = j - 1 ;
          // std::cout << "循环次数: " << times << " 次，从 0x" << std::hex 
          //           << beginInst.offset << " 到 0x" << endInst.offset << std::endl;
          const unsigned tmp = times * 3 - times * 2 - 4; 
          theorTotalBytes += tmp * 4;
          break;
        } else {
          accept = false;
          break;
        }
      } else if (insns[j].op == "LUI" && insns[j + 1].op == "ADDI" && insns[j + 2].op == "JAL"){
        // 边界条件：遇到了其他lui高位的imm，直接取下一个LUI
        if (IH::getRd(insns[j]) != IH::getRs1(insns[j + 1])){
          // 边界条件，lui的rd应该为addi的rs1
          accept = false;
          break;
        } else {
          // std::cout << "Cur offset: " << std::hex << insns[j].offset
          //           << "\tLUI\t" << insns[j].rd << "," << insns[j].imm
          //           << ":\t → " << times << "\n";
          times++;
          j += 3;
          // 取潜在的下一组
        }
      } else {
        accept = false;
        break;
      }
    }
    /** 第四步：计算所需的GP和新GP插入点和旧GP删除点  */ 
    if (accept){
      /** 4.0：计算需要的新GP */
      const Imm Hi20 = IH::getImm(I_1);
      const Imm Low12 = IH::getImm(I_2);
      unsigned jumpPoint = (Hi20 << 12) + Low12;
      const unsigned __new_GP_ = jumpPoint;
      std::cout << "新的GP设置为: " << std::hex << __new_GP_ ;
      std::cout << "\t范围：" << insns[beginIdx].offset << " → " << insns[endIdx].offset;
      char* startAddr = IH::getAddr(insns[beginIdx]);
      char* lastAddr  = IH::getAddr(insns[endIdx]);
      // NOTE 这一步之前都是对的
      /** 4.1：收集该区域内指令，方便后续予以覆盖 */
      // 加4是因为包括lastAddr所在的指令
      const unsigned instCnt  = endIdx - beginIdx + 1; // 指令数 
      // 根据公式收集所有指令
      std::deque<RV32Inst> instBatch;
      for (int k = beginIdx; k <= endIdx; k++)
        instBatch.push_back(insns[k]);
      // 此时instBatch存储lui, addi, jal组
      /** 4.2：在startAddr开始8字节填入lui+addi加载新GP */
      uint32_t newGPHi20 = (__new_GP_ >> 12) & 0xFFFFF;
      uint32_t newGPLow12 = __new_GP_ & 0xFFF;
      // LUI指令，加载高20位到gp; 3是gp寄存器
      uint32_t luiInst = (newGPHi20 << 12) | (3 << 7) | 0x37;
      // ADDI指令，添加低12位到gp
      uint32_t addiInst = (newGPLow12 << 20) | (3 << 15) | (3 << 7) | 0x13;
      std::memcpy(startAddr, &luiInst, sizeof(uint32_t));
      std::memcpy(startAddr + 4, &addiInst, sizeof(uint32_t));

      /** 4.3：在lastAddr结束8字节填入lui+addi加载旧GP */
      uint32_t oldGPHi20 = (__GP_Origin_ >> 12) & 0xFFFFF;
      uint32_t oldGPLow12 = __GP_Origin_ & 0xFFF;
      luiInst = (oldGPHi20 << 12) | (3 << 7) | 0x37; 
      addiInst = (oldGPLow12 << 20) | (3 << 15) | (3 << 7) | 0x13;
      std::memcpy(lastAddr - 4, &luiInst, sizeof(uint32_t));
      std::memcpy(lastAddr , &addiInst, sizeof(uint32_t));

      /** 4.4：对新内存每条addi指令：比如 addi a1, a1, 1000 修改为 addi a1, gp, off */
      // 对每条addi
      // 修改每个addi命令
      std::cout << "\n指令数: " << std::dec << endIdx - beginIdx + 1 << "条\n";
      for (unsigned idx = 0; idx < instCnt; idx++) {
        if (IH::isADDI(instBatch[idx])){
          RV32Inst &TB = instBatch[idx];
          uint32_t inst = TB.raw;
          // 计算原立即数与新GP的插值
          RV32Inst &TA = instBatch[idx - 1];
          const auto immNew = ((IH::getImm(TA) << 12) + IH::getImm(TB)) - __new_GP_;
          const auto originRd = IH::getRd(TB);
          TB.raw = (immNew << 20) | (3 << 15) | (originRd << 7) | 0x13;
          std::cout << std::hex << "\t修改: " << idx << " \t" << instBatch[idx].offset << "\t新立即数: " << immNew << "\t 0x" << TB.raw;
        } else if (!IH::isLUI(instBatch[idx])){
          // JAL的跳转值需要修正，即计算原始位置和新位置插值
          const auto originRelaPlace = idx + 1; // 代表第x条
          const auto adjustRelaPlace = 2 + idx / 3 * 2 + 2;
          // NOTE 断言前跳, 获取JAL指令
          RV32Inst &T = instBatch[idx];
          // offset[20|10:1|11|19:12] | rd | 1101111
          uint32_t raw = T.raw;
          Imm originImm = IH::getImm(instBatch[idx]);
          if (originImm < 0)
            std::cout << "\tJAL跳转offset = -0x" << std::hex << -originImm;
          else
            std::cout << "\tJAL跳转offset =  0x" << std::hex << originImm;
          // 修改立即数，编码重新写入T.raw
          std::cout << "调整值:" << (adjustRelaPlace - originRelaPlace) * 0x4 << std::endl;
          originImm -= (adjustRelaPlace - originRelaPlace) * 4;
          int32_t encodedImm = 
              ((originImm & 0x100000) << 11) |  // offset[20] -> bit 31
              ((originImm & 0x7FE) << 20) |    // offset[10:1] -> bit 30:21
              ((originImm & 0x800) << 9) |     // offset[11] -> bit 20
              (originImm & 0xFF000);           // offset[19:12] -> bit 19:12
          T.raw = encodedImm | (IH::getRd(instBatch[idx]) << 7) | 0x6f;
        }
      }

      /** 4.5 计算需要填充的nop数量 */
      // 因为指令是lui,addi,jal对，所以能被3整除
      const unsigned instTrunk = instCnt / 3;
      const unsigned instNop = instCnt - instTrunk * 2 - 4;
      std::cout << std::dec << "基础信息:组合数 = " << instTrunk << " 空指令数 = " << instNop << std::endl;
      /** 4.6：对所有指令，如果不是lui，则移动回startAddr + 8开始的地址，按序摆放 */
      int tmp = 0;
      for (unsigned idx = 0; idx < instCnt; idx++) { 
        // 移动的位置为 insns[i + 2] 
        RV32Inst &T = instBatch[idx];
        if (!IH::isLUI(instBatch[idx])){
          std::cout << std::dec << "\t移动: " << idx << " \t回 " << std::hex <<insns[beginIdx + tmp + 2].offset << "\traw = 0x" << insns[beginIdx + tmp + 2].raw << "\n";
          const uint32_t writeVal = T.raw;
          char* writeAddr = IH::getAddr(insns[beginIdx + tmp++ + 2]); 
          uint32_t* addi_jal  = reinterpret_cast<uint32_t*>(writeAddr);
          *addi_jal = writeVal;
        }
      }

      /** 4.7：对旧内存其余空位补nop指令 */
      // 循环填充NOP指令
      for (unsigned k = 0; k < instNop; k++) { 
        uint32_t* inst = reinterpret_cast<uint32_t*>(insns[beginIdx + tmp++ + 2].address);
        if (k == instNop - 1) // NOTE 实验性!
          *inst = 0x00000013;
        else
          *inst = 0b00000000000000001110000010110011; // NOP指令编码
      }
    }
    i = j;
    if (accept) break;
  }
  std::cout << "理论删除字节数：" << theorTotalBytes << std::endl;
  ObjReader::synchronizeInsts("afterDyGP");
  // BUG 跳转表要更新
  return true;
}
	phWorker.doDynamicGPRelaxing();
	phWorker.doNopCleaning();
```


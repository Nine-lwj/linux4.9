# SATA控制器、PHY、设备三者交互详细流程

## 🏗️ **三层架构概览**

```mermaid
graph TD
    A[Linux SATA驱动层] --> B[AHCI控制器]
    B --> C[SATA PHY]
    C --> D[SATA设备]
    
    A1[libata-core.c] --> A
    A2[libahci.c] --> A
    A3[ahci.c] --> A
    
    B1[SCR寄存器] --> B
    B2[FIS处理] --> B
    B3[DMA引擎] --> B
    
    C1[物理信号] --> C
    C2[链路训练] --> C
    C3[8b/10b编码] --> C
    
    D1[SATA磁盘] --> D
    D2[SATA SSD] --> D
    
    style A fill:#e3f2fd
    style B fill:#e8f5e8
    style C fill:#fff3e0
    style D fill:#fce4ec
```

## 📋 **完整交互流程图**

### **阶段1：系统启动和驱动初始化**

```mermaid
sequenceDiagram
    participant Kernel as Linux内核
    participant Driver as AHCI驱动
    participant Controller as AHCI控制器
    participant PHY as SATA PHY
    participant Device as SATA设备
    
    Note over Kernel,Device: 系统启动阶段
    
    Kernel->>Driver: 1. pci_register_driver()
    Driver->>Driver: 2. ahci_init_one()
    Driver->>Controller: 3. ahci_save_initial_config()
    Driver->>Controller: 4. ahci_reset_controller()
    
    Note over Controller: 控制器复位
    Controller->>PHY: 5. 硬件复位信号
    PHY->>PHY: 6. PHY内部初始化
    
    Driver->>Driver: 7. ahci_host_activate()
    Driver->>Driver: 8. ata_host_start()
    
    Note over Driver: 为每个端口创建工作队列
    loop 每个SATA端口
        Driver->>Driver: 9. ata_port_probe()
        Driver->>Driver: 10. ata_port_schedule_eh()
    end
```

### **阶段2：端口探测和PHY链路建立**

```mermaid
sequenceDiagram
    participant EH as 错误处理线程
    participant Driver as AHCI驱动
    participant Controller as AHCI控制器  
    participant PHY as SATA PHY
    participant Device as SATA设备
    
    Note over EH,Device: 端口探测阶段
    
    EH->>EH: 1. ata_eh_autopsy()
    EH->>EH: 2. ata_eh_reset()
    EH->>Driver: 3. ahci_hardreset()
    
    Note over Driver,PHY: 硬重置流程开始
    Driver->>Driver: 4. sata_link_hardreset()
    Driver->>Controller: 5. sata_scr_write(SCR_CONTROL, 0x301)
    Controller->>PHY: 6. 设置DET=1 (PHY重置)
    
    Note over PHY: PHY开始发送COMRESET
    PHY->>Device: 7. COMRESET信号 (320ns低电平)
    Device->>Device: 8. 检测COMRESET
    Device->>PHY: 9. 发送COMWAKE (106.7ns脉冲串)
    
    Driver->>Driver: 10. ata_msleep(1ms)
    Driver->>Driver: 11. sata_link_resume()
    Driver->>Controller: 12. sata_scr_write(SCR_CONTROL, 0x300)
    Controller->>PHY: 13. 设置DET=0 (正常模式)
    
    Note over PHY: PHY检测COMWAKE
    PHY->>PHY: 14. COMWAKE检测和验证
    PHY->>Device: 15. 发送COMINIT确认
    Device->>PHY: 16. 发送ALIGN原语
    PHY->>Controller: 17. 报告DET=3 (链路建立)
    
    Driver->>Driver: 18. ata_msleep(200ms)
    Driver->>Driver: 19. sata_link_debounce()
    Driver->>Controller: 20. sata_scr_read(SCR_STATUS)
    Controller->>Driver: 21. 返回SStatus (DET状态)
    
    alt DET=3 (正常情况)
        Driver->>Driver: 22. ata_phys_link_offline() 返回false
        Driver->>Driver: 23. 进入软重置阶段
    else DET=1 (您的情况)
        Driver->>Driver: 22. ata_phys_link_offline() 返回true
        Driver->>Driver: 23. goto out (跳过软重置)
    end
```

### **阶段3：设备识别和软重置**

```mermaid
sequenceDiagram
    participant Driver as AHCI驱动
    participant Controller as AHCI控制器
    participant PHY as SATA PHY
    participant Device as SATA设备
    
    Note over Driver,Device: 软重置和设备识别
    
    Driver->>Driver: 1. ahci_do_softreset()
    
    Note over Driver: 准备第一个FIS
    Driver->>Driver: 2. ata_tf_init()
    Driver->>Driver: 3. tf.ctl |= ATA_SRST
    Driver->>Driver: 4. ata_tf_to_fis()
    Driver->>Driver: 5. ahci_exec_polled_cmd()
    
    Note over Controller: AHCI控制器处理FIS
    Driver->>Controller: 6. ahci_fill_cmd_slot()
    Driver->>Controller: 7. writel(1, PORT_CMD_ISSUE)
    Controller->>Controller: 8. 构建FIS数据包
    Controller->>PHY: 9. 发送FIS到PHY
    
    Note over PHY: PHY编码和传输
    PHY->>PHY: 10. 8b/10b编码
    PHY->>Device: 11. 串行数据传输
    Device->>Device: 12. 8b/10b解码
    Device->>Device: 13. 解析FIS命令
    
    alt 设备正常响应
        Device->>Device: 14. 执行软重置
        Device->>PHY: 15. 发送D2H Register FIS
        PHY->>PHY: 16. 8b/10b编码
        PHY->>Controller: 17. 串行数据接收
        Controller->>Controller: 18. FIS解析
        Controller->>Driver: 19. 中断通知
        Driver->>Driver: 20. 检查命令完成状态
    else 设备无响应 (您的情况)
        Note over Device: 设备无法正确处理FIS
        Driver->>Driver: 14. ata_wait_register() 超时
        Driver->>Driver: 15. 返回 "1st FIS failed"
    end
```

### **阶段4：设备识别和初始化**

```mermaid
sequenceDiagram
    participant Driver as AHCI驱动
    participant Controller as AHCI控制器
    participant PHY as SATA PHY
    participant Device as SATA设备
    
    Note over Driver,Device: 设备IDENTIFY过程
    
    Driver->>Driver: 1. ata_wait_after_reset()
    Driver->>Driver: 2. ahci_check_ready()
    Driver->>Controller: 3. 读取Task File状态
    
    alt 设备就绪
        Driver->>Driver: 4. ahci_dev_classify()
        Driver->>Driver: 5. ata_dev_read_id()
        
        Note over Driver: 发送IDENTIFY命令
        Driver->>Driver: 6. ata_exec_internal()
        Driver->>Controller: 7. 构建IDENTIFY FIS
        Controller->>PHY: 8. 发送IDENTIFY命令
        PHY->>Device: 9. 传输命令
        
        Device->>Device: 10. 准备IDENTIFY数据
        Device->>PHY: 11. 发送512字节数据
        PHY->>Controller: 12. 接收数据
        Controller->>Driver: 13. DMA传输到内存
        
        Driver->>Driver: 14. 解析设备信息
        Driver->>Driver: 15. ata_dev_configure()
        Driver->>Driver: 16. 设置传输模式
        
        Note over Driver: 设备初始化完成
        Driver->>Driver: 17. ata_scsi_scan_host()
        Driver->>Driver: 18. 创建/dev/sdX设备节点
        
    else 设备未就绪 (您的情况)
        Driver->>Driver: 4. 超时等待
        Driver->>Driver: 5. 返回 "device not ready"
    end
```

## 🔧 **关键函数调用链详解**

### **硬重置阶段的关键函数**

```c
// 主调用链
ata_eh_reset()
  └── ahci_hardreset()
      └── sata_link_hardreset()
          ├── sata_scr_write(SCR_CONTROL, 0x301)  // 发送COMRESET
          ├── ata_msleep(1)                       // 等待1ms
          └── sata_link_resume()
              ├── sata_scr_write(SCR_CONTROL, 0x300)  // 正常模式
              ├── ata_msleep(200)                     // 等待PHY稳定
              └── sata_link_debounce()
                  └── sata_scr_read(SCR_STATUS)      // 检查DET状态

// 状态检查函数
ata_phys_link_offline()
  └── ata_sstatus_online()
      └── return (sstatus & 0xf) == 0x3;  // 您的PHY这里返回false
```

### **软重置阶段的关键函数**

```c
// 软重置调用链
ahci_do_softreset()
  ├── ata_tf_init()                    // 初始化Task File
  ├── ata_tf_to_fis()                  // 转换为FIS格式
  └── ahci_exec_polled_cmd()
      ├── ahci_fill_cmd_slot()         // 填充命令槽
      ├── writel(PORT_CMD_ISSUE)       // 发送命令
      └── ata_wait_register()          // 等待完成
```

## 🎯 **您的1.5G PHY具体卡住的位置**

### **硬件交互层面**

```mermaid
graph TD
    A[AHCI控制器写入SCR_CONTROL=0x301] --> B[PHY接收到重置命令]
    B --> C[PHY发送COMRESET到设备]
    C --> D[设备检测COMRESET]
    D --> E[设备发送COMWAKE响应]
    E --> F{PHY能否正确检测COMWAKE?}
    
    F -->|是| G[PHY设置内部状态DET=3]
    F -->|否| H[PHY保持DET=1状态]
    
    G --> I[控制器读取SCR_STATUS返回DET=3]
    H --> J[控制器读取SCR_STATUS返回DET=1]
    
    I --> K[软件认为链路在线]
    J --> L[软件认为链路离线]
    
    style F fill:#ffcccc
    style H fill:#ffaaaa
    style J fill:#ffcccc
    style L fill:#ffaaaa
```

### **软件层面的判断逻辑**

```c
// 您的PHY在这个流程中失败：
sata_link_hardreset()
  └── sata_link_resume()           // PHY应该在这里转换到DET=3
      └── sata_link_debounce()     // 但实际上保持在DET=1
          └── 返回DET=1到上层
              └── ata_phys_link_offline()
                  └── ata_sstatus_online(SStatus=1)
                      └── return false  // 认为离线!
                          └── 跳过软重置阶段
```

## 💡 **问题本质和解决方向**

### **问题本质**
1. **PHY硬件问题**：无法正确检测或处理设备的COMWAKE信号
2. **软件判断过严**：只认DET=3为在线，不允许DET=1进入软重置
3. **错失测试机会**：因为DET=1被拒绝，无法测试数据通道是否正常

### **解决策略**
1. **短期workaround**：修改软件逻辑，允许DET=1进入软重置测试
2. **中期修复**：调整PHY参数，提高COMWAKE检测能力
3. **长期解决**：根据测试结果决定是否需要硬件重新设计

这个流程图清晰地展示了您的PHY在整个交互过程中的具体失败点，以及各个组件之间的详细交互关系。
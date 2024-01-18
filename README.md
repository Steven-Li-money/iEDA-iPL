## iPL源码分析
流程：全局布局--->合法化--->详细布局

### 执行iPL点工具的Tcl脚本流程：
- 加载工艺库，加载数据
- def设计文件的读入
- 具体功能模块
- netlist存储
- 做一个report

### 执行脚本
```bash
./iEDA -script ./script/iPL_script/run_iPL.tcl 
```
#### tcl脚本解析
##### run_iPL.tcl
- init flow config
  - flow_init -config ./iEDA_config/flow_config.json
- read db config
  - db_init -config ./iEDA_config/db_default_config.json
- reset data path
  - source ./script/DB_script/db_path_setting.tcl
- reset lib
  - source ./script/DB_script/db_init_lib.tcl
- reset sdc
  - source ./script/DB_script/db_init_sdc.tcl
- read lef
  - source ./script/DB_script/db_init_lef.tcl
- read def
  - def_init -path ./result/iTO_fix_fanout_result.def
- run Placer
  - run_placer -config ./iEDA_config/pl_default_config.json
- save def
  - def_save -path ./result/iPL_result.def
- save netlist
  - netlist_save -path ./result/iPL_result.v -exclude_cell_names {}
- report
  - report_db -path "./result/report/pl_db.rpt"
- Exit
  - flow_exit
### tcl脚本与c++交互实现流程（以run_placer命令为例）
#### 一、脚本命令注册
- iEDA/src/interface/tcl/tcl_ipl/tcl_register_pl.h脚本任务注册
- run_placer，通过c++代码注册到TCL环境中，相应的 C++ 模块应当已经被加载和初始化，TCL 解释器会调用相应的函数来执行这个命令
- registerTclCmd(CmdPlacerAutoRun, "run_placer")将 run_placer 命令与 CmdPlacerAutoRun 函数关联起来。这意味着当 run_placer 命令在 TCL 脚本中被调用时，CmdPlacerAutoRun 函数将被执行
#### 二、实现CmdPlacerAutoRun函数
- iEDA/src/interface/tcl/tcl_ipl/tcl_ipl.cpp中实现了CmdPlacerAutoRun函数
- 构造函数 CmdPlacerAutoRun::CmdPlacerAutoRun： 这个构造函数初始化 CmdPlacerAutoRun 对象。它接受一个命令名（cmd_name），并调用基类 TclCmd 的构造函数。这表明 CmdPlacerAutoRun 是 TclCmd 的子类，可能是用于执行 TCL 命令的通用框架。在构造函数中，还创建了一个 TclStringOption 对象，用于处理命令行选项（这里是文件名选项）。这个选项被添加到命令中，允许命令处理类似 -config 这样的参数
- check 方法： check 方法似乎用于验证命令是否有有效的输入或参数。这里，它检查 file_name_option 是否存在。如果不存在，会记录一个致命错误。这是命令执行前的一种安全检查
- exec 方法： 这是命令的主要执行函数。它首先调用 check 来确保所有必要的参数都在位。然后，它获取配置文件的名称，并将其传递给 autoRunPlacer 方法（可能是某个布局工具的一部分）。如果 autoRunPlacer 成功执行，它会输出一条成功信息
#### 三、实现autoRunPlacer函数
- iEDA/src/platform/tool_manager/tool_manager.cpp中实现了autoRunPlacer 方法
- ToolManager::autoRunPlacer 方法的作用是管理布局工具（Placer）的整个运行流程。它接收一个字符串参数 config，这个参数大概是指向配置文件的路径或包含配置数据。方法的主要步骤如下：
  - 初始化 Placer：通过调用 plInst->initPlacer(config)，使用提供的配置数据初始化 Placer 实例。这可能涉及设置布局参数、加载必要的数据文件等准备工作
  - 运行布局过程：接着调用 plInst->runPlacement() 来实际执行布局过程。这可能是一个复杂的过程，涉及算法执行、资源分配、优化等步骤。runPlacement 方法返回一个布尔值，表示布局过程是否成功
  - 清理：完成布局后，调用 plInst->destroyPlacer() 来进行清理工作。这可能包括释放资源、保存结果文件等
  - 返回状态：最后，方法返回一个布尔值 flag，指示布局过程是否成功完成
#### 四、实现runPlacement
- 检查和初始化：
  - 方法首先检查布局数据库（PlacerDB）是否已经启动，使用 iPLAPIInst.isPlacerDBStarted() 进行检查
  - 如果布局数据库未启动，它会通过调用 this->initPlacer("") 进行初始化。这里的空字符串参数可能表示默认配置或无需额外配置
  - 如果布局数据库已经启动，它则调用 iPLAPIInst.updatePlacerDB() 更新布局数据库，可能是为了同步最新的数据或状态
- 执行布局流程：使用 iPLAPIInst.runFlow() 执行实际的布局流程。这可能涉及一系列复杂的布局算法和过程，但具体细节并未在代码段中给出
- 收集和记录统计数据：
  - 创建一个 ieda::Stats 对象（可能用于收集和跟踪运行统计）
  - 通过 flowConfigInst->add_status_runtime(stats.elapsedRunTime()) 记录运行时间
  - 通过 flowConfigInst->set_status_memory(stats.memoryDelta()) 记录内存使用情况
  - 返回结果：方法返回 true，表明布局过程完成
#### 五、实现runflow
- iEDA/src/operation/iPL/api/PLAPI.cc中实现runflow等
- PLAPI::runFlow 被 PlacerIO::runPlacement 调用，作为执行具体布局任务的核心流程。这个方法包含了布局过程的不同阶段和步骤：
  - 执行全局布局（Global Placement, GP）：
    - 通过调用 runGP() 方法进行全局布局。全局布局通常涉及在芯片上大致安排电路元件的位置
  - 打印半周长线长信息（Half Perimeter Wire Length, HPWL）：
    - 使用 printHPWLInfo() 打印当前布局的半周长线长信息，这是衡量布局质量的一个重要指标
  - 运行缓冲插入（可选）：
    - 如果配置允许，执行 runBufferInsertion() 来进行缓冲插入优化，并再次打印 HPWL 信息
  - 执行合法化布局（Legalization, LG）：
    - 通过 runLG() 进行合法化布局，确保布局结果符合设计规则
  - 执行详细布局（Detailed Placement, DP）：
    - 使用 runDP() 进行详细布局，这是布局过程的最后阶段，负责优化电路元件的确切位置
  - 生成报告和写回源数据库：
    - 使用 reportPLInfo() 生成布局过程的报告
    - 调用 writeBackSourceDataBase() 将布局结果写回源数据库
- 整体来看，PLAPI::runFlow 方法执行了一系列布局步骤，从全局布局到详细布局，包括可能的优化步骤（如缓冲插入）。这个过程是布局任务中最为核心和复杂的部分，涉及到多个布局算法和技术的应用。它被 PlacerIO::runPlacement 方法调用，后者作为更高层次的接口，负责整个布局任务的启动和完成
#### 六、执行实际代码
- 后面就是执行iEDA/src/operation/iPL/source/module/legalizer/Legalizer.cc等代码了

### 参数配置
参考iEDA_config/pl_default_config.json: `./scripts/design/sky130_gcd/iEDA_config/pl_default_config.json`

| JSON参数                                      | 功能说明                                                                                                                    | 参数范围                     | 默认值        |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ---------------------------- | ------------- |
| is_max_length_opt                             | 是否开启最大线长优化                                                                                                        | [0,1]                        | 0             |
| max_length_constraint                         | 指定最大线长                                                                                                                | [0-1000000]                  | 1000000       |
| is_timing_aware_mode                          | 是否开启时序模式                                                                                                            | [0,1]                        | 0             |
| ignore_net_degree                             | 忽略超过指定pin个数的线网                                                                                                   | [10-10000]                   | 100           |
| num_threads                                   | 指定的CPU线程数                                                                                                             | [1-64]                       | 8             |
| [GP-Wirelength] init_wirelength_coef          | 设置初始线长系数                                                                                                            | [0.0-1.0]                    | 0.25          |
| [GP-Wirelength] reference_hpwl                | 调整密度惩罚的参考线长                                                                                                      | [100-1000000]                | 446000000     |
| [GP-Wirelength] min_wirelength_force_bar      | 控制线长边界                                                                                                                | [-1000-0]                    | -300          |
| [GP-Density] target_density                   | 指定的目标密度                                                                                                              | [0.0-1.0]                    | 0.8           |
| [GP-Density] bin_cnt_x                        | 指定水平方向上Bin的个数                                                                                                     | [16,32,64,128,256,512,1024]  | 512           |
| [GP-Density] bin_cnt_y                        | 指定垂直方向上Bin的个数                                                                                                     | [16,32,64,128,256,512,1024]  | 512           |
| [GP-Nesterov] max_iter                        | 指定最大的迭代次数                                                                                                          | [50-2000]                    | 2000          |
| [GP-Nesterov] max_backtrack                   | 指定最大的回溯次数                                                                                                          | [0-100]                      | 10            |
| [GP-Nesterov] init_density_penalty            | 指定初始状态的密度惩罚                                                                                                      | [0.0-1.0]                    | 0.00008       |
| [GP-Nesterov] target_overflow                 | 指定目标的溢出值                                                                                                            | [0.0-1.0]                    | 0.1           |
| [GP-Nesterov] initial_prev_coordi_update_coef | 初始扰动坐标时的系数                                                                                                        | [10-10000]                   | 100           |
| [GP-Nesterov] min_precondition                | 设置precondition的最小值                                                                                                    | [1-100]                      | 1             |
| [GP-Nesterov] min_phi_coef                    | 设置最小的phi参数                                                                                                           | [0.0-1.0]                    | 0.95          |
| [GP-Nesterov] max_phi_coef                    | 设置最大的phi参数                                                                                                           | [0.0-1.0]                    | 1.05          |
| [BUFFER] max_buffer_num                       | 指定限制最大buffer插入个数                                                                                                  | [0-1000000]                  | 35000         |
| [BUFFER] buffer_type                          | 指定可插入的buffer类型名字                                                                                                  | 工艺相关                     | 列表[...,...] |
| [LG] max_displacement                         | 指定单元的最大移动量                                                                                                        | [10000-1000000]              | 50000         |
| [LG] global_right_padding                     | 指定单元间的间距（以Site为单位）                                                                                            | [0,1,2,3,4...]               | 1             |
| [DP] max_displacement                         | 指定单元的最大移动量                                                                                                        | [10000-1000000]              | 50000         |
| [DP] global_right_padding                     | 指定单元间的间距（以Site为单位）                                                                                            | [0,1,2,3,4...]               | 1             |
| [Filler] first_iter                           | 指定第一轮迭代使用的Filler                                                                                                  | 工艺相关                     | 列表[...,...] |
| [Filler] second_iter                          | 指定第二轮迭代使用的Filler                                                                                                  | 工艺相关                     | 列表[...,...] |
| [Filler] min_filler_width                     | 指定Filler的最小宽度（以Site为单位）                                                                                        | 工艺相关                     | 1             |
| [MP] fixed_macro                              | 指定固定的宏单元 (string macro_name)                                                                                        | 设计相关                     | 列表[...,...] |
| [MP] fixed_macro_coordinate                   | 指定固定宏单元的位置坐标（int location_x, int location_y）                                                                  | 设计相关                     | 列表[...,...] |
| [MP] blockage                                 | 指定宏单元阻塞矩形区域，宏单元应该避免摆放在该区域（int left_bottom_x, int left_bottom_y, int right_top_x, int right_top_y) | 设计相关                     | 列表[...,...] |
| [MP] guidance_macro                           | 指定指导摆放宏单元，每个宏单元可以设置期望摆放的区域 (string macro_name)                                                    | 设计相关                     | 列表[...,...] |
| [MP] guidance                                 | 指定对应宏单元的指导摆放区域（int left_bottom_x, int left_bottom_y, int right_top_x, int right_top_y）                      | 设计相关                     | 列表[...,...] |
| [MP] solution_type                            | 指定解的表示方式                                                                                                            | ["BStarTree","SequencePair"] | "BStarTree"   |
| [MP] perturb_per_step                         | 指定模拟退火每步扰动次数                                                                                                    | [10-1000]                    | 100           |
| [MP] cool_rate                                | 指定模拟退火温度冷却率                                                                                                      | [0.0-1.0]                    | 0.92          |
| [MP] parts                                    | 指定标准单元划分数（int)                                                                                                    | [10-100]                     | 66            |
| [MP] ufactor                                  | 指定标准单元划分不平衡值 (int)                                                                                              | [10-1000]                    | 100           |
| [MP] new_macro_density                        | 指定虚拟宏单元密度                                                                                                          | [0.0-1.0]                    | 0.6           |
| [MP] halo_x                                   | 指定宏单元的halo（横向）                                                                                                    | [0-1000000]                  | 0             |
| [MP] halo_y                                   | 指定宏单元的halo（纵向）                                                                                                    | [0-1000000]                  | 0             |
| [MP] output_path                              | 指定输出文件路径                                                                                                            |                              | "./result/pl" |

### iPL目前完成的功能有
- 数据读入支持从iDB提取和处理相关布局相关数据；配置文件支持json读入
- 满足布局流程需求
  - - 全局布局：支持HPWL、STWL、WAWL线长和电场密度（梯度）评估，使用Nesterov法布局
  - - 合法化：目的是消除单元互相不重叠，使用Abacus方法，支持完整和增量式合法化模式
  - - 详细布局：目的是进行指标（线长、时序、可布线性）的局部优化，支持单元单行shift（行内最优化布局）
  - - Filler填充：支持指定filler类型填充布局区域的空白
  - - 布局检查：单元放置在布局区域内、对齐Row/Site，对齐Power rail、单元是否有重叠


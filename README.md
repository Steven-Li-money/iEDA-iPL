# iEDA-iPL
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


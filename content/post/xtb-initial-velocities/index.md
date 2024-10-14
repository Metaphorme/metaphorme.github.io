---
title: "xTB 速度初始化方法"
date: 2024-04-04T17:20:00+08:00
tags: ['chemistry', 'Molecular Dynamics']
---

浅浅探讨一下 xTB 速度初始化方法。

<!--more-->

很好奇 [xTB](https://github.com/grimme-lab/xtb) 进行分子动力学模拟时如何进行速度初始化，遂拔一下代码，记录在本文。

由于我缺少 Fortran 调试器（实际上 xTB 根本没法在 arm 架构的 Mac 进行分子动力学模拟😭），并且没有 Fortran 经验，导致本文的疏漏较多，还请指教。

## 部分源码

摘自：[https://github.com/grimme-lab/xtb/blob/main/src/dynamic.f90](https://github.com/grimme-lab/xtb/blob/main/src/dynamic.f90)

```Fortran
!cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc
! initialize velocities uniformly
!cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc

subroutine mdinitu(n,iat,velo,mass,Ekin)
   use xtb_setparam
   implicit none
   integer n
   real(wp) velo(3,n)
   integer iat(n)
   real(wp) mass(n)
   real(wp) Ekin
   real x,ranf

   real(wp) eperat,v,f,t,edum,f2
   integer i,j
   integer, allocatable :: iseed(:)

   call initrand

   eperat=Ekin/(3.0*n)

   do i=1,n
      f2=1
      if(iat(i).eq.1) f2=2
      v=sqrt(2*eperat/mass(i))
      f=1.0d0
      call random_number(x)
      if(x.gt.0.5)f=-1.0d0
      velo(1,i)=v*f*f2
      f=1.0d0
      call random_number(x)
      if(x.gt.0.5)f=-1.0d0
      velo(2,i)=v*f*f2
      f=1.0d0
      call random_number(x)
      if(x.gt.0.5)f=-1.0d0
      velo(3,i)=v*f*f2
   enddo

   call ekinet(n,velo,mass,edum)

   t=edum/(0.5*3*n*0.316681534524639E-05)

end subroutine mdinitu
```

## 调用示例

摘自：[https://github.com/grimme-lab/xtb/blob/main/src/dynamic.f90](https://github.com/grimme-lab/xtb/blob/main/src/dynamic.f90)

```Fortran
   molmass=0
   do i=1,mol%n
      molmass=molmass+atmass(i)
      mass(i)=atmass(i)*amutoau
   enddo
   tmass=molmass*amutoau
   do i=1,mol%n
      if(mol%at(i).eq.1.and.set%md_hmass.gt.0) then
         mass(i)=dble(set%md_hmass)*amutoau
         ! k=nbo(1,i) ! atom to which H is bonded
         ! mass(k)=mass(k)-mass(i)+ams(1)*amutoau ! reduce by the increase H mass
      endif
   enddo
   molmass=molmass*amutokg

   nfreedom = 3._wp*real(mol%n,wp)  ! 注：可以理解为 nfreedom = 原子数目 * 3
   if(set%mdrtrconstr ) nfreedom = nfreedom - 6.0d0
   if(set%shake_md    ) nfreedom = nfreedom - dble(ncons)
   if(zconstr.eq.1) nfreedom = nfreedom - dble(iatf1)  ! fragment 1 in Z plane
   write(*,'('' # deg. of freedom  :'',i6  )')idint(nfreedom)

   Tinit=Tsoll  ! 注：实践发现 Tsoll 是 xtb 配置文件中 temp 量。
   if (equi) Tinit=0.3*Tsoll   ! slow equi 注：“slow equi"可能是指"slow equilibration”（慢速平衡）的简称
   
   if(.not.restart)then
      f=1
      if(.not.thermostat) f = 2
      edum=f*Tinit*0.5*kB*nfreedom
      ! 注：Tinit 暂时理解为初始温度
      ! 注：kB 在 src/mctc/mctc_constants.f90 定义为 3.166808578545117e-06，指的是 Boltzmann constant in Eh/K
      ! 注：nfreedom 暂时理解为原子数目 * 3
      
      call mdinitu(mol%n,mol%at,velo,mass,edum)
   else
      call rdmdrestart(mol%n,mol%xyz,velo)
   endif
```

## 输入量
| 变量 | 类型       |
| ---- | ---------- |
| n    | 整数       |
| iat  | n元素数组  |
| velo | n行3列数组 |
| mass | n元素数组  |
| Ekin | 浮点数     |


### iat
mol 是 TMolecule 类型的。TMolecule 在 https://github.com/grimme-lab/xtb/blob/main/src/type/molecule.f90 中定义。
其中 iat 的定义如下：
```Fortran
      !> Ordinal numbers
      integer, allocatable :: at(:)
```
原子序数，在 https://github.com/grimme-lab/xtb/blob/main/src/cqpath.f90 定义。

### mass
在 https://github.com/grimme-lab/xtb/blob/main/src/param/atomicmass.f90 定义，单位为 u。
实际调用时，使用 amutoau * [(u)]

amutoau 在 https://github.com/grimme-lab/xtb/blob/main/src/mctc/convert.f90 定义，值为 amutokg * kgtome = 1.660539040e-27 * 1 / 9.10938356e-31 （转换为电子质量单位 Me）。

### Ekin

thermostat 是布尔类型变量，是 xTB 配置文件中 nvt 量，默认 true。

## 讨论

1. xTB 进行速度初始化的大致步骤
   1. 如果 thermostat （恒温器，是 xTB 配置文件中 [nvt 参数](https://xtb-docs.readthedocs.io/en/latest/md.html#parameters)，默认 true）定义为 False，则 f = 2，否则 f = 1；
   2. Tsoll 在实验中发现和 [temp 参数](https://xtb-docs.readthedocs.io/en/latest/md.html#parameters)相等，如果 equi 为 True，则 Tinit=0.3*Tsoll；否则 Tinit=Tsoll；
   3. nfreedom 在大多数情况下定义为 3*原子数，其他情况请见[调用示例](#调用示例)；
   4. 将以上值带入公式：edum=f * Tinit * 0.5 * kB * nfreedom（注：公式来源不明），得到 edum；
   5. mol%n 是体系原子数；
   6. mol%at 是体系原子序数列表；
   7. velo 是仅占用内存空间的n行3列数组，用于保存速度；
   8. mass 是n元素数组，用于保存原子质量（单位：电子质量单位 Me）（计算公式：原子质量(u) * 1.660539040e-27 * 1 / 9.10938356e-31）;
   9. 之前计算的 edum 在 mdinitu 函数的变量名为 Ekin，带入公式 eperat=Ekin / (3.0 * n)，得到 eperat；
   10. 如果原子是 H，则 f2 = 2，否则 f2 = 1；
   11. v=sqrt(2 * eperat / 此原子mass)；
   12. 令 f=1.0；
   13. 生成 [0, 1) 随机数 x；
   14. 如果 x > 0.5，则 f = -1；
   15. Vx = v * f * f2
   16. 回到步骤 xii，直到生成此原子的 Vx，Vy，Vz；
   17. 回到步骤 x，直到生成所有原子的速度。
2. xTB 初始化速度特点：
   1. 为体系的每个原子提供**相同的初始动能**（能量均分定理）；
   2. 相同类型原子 Vx，Vy，Vz 的**绝对值相同，正负性服从均匀分布，而非麦克斯韦-玻尔兹曼分布**。
3. 疑惑：
   1. edum=f * Tinit * 0.5 * kB * nfreedom 公式来源（可能与能量均分定理有关）；
   2. eperat=Ekin / (3.0 * n) 公式来源（可能与能量均分定理有关）；
   3. 速度单位是什么？

## Python 实现

理清 xTB 初始化速度的原理后，我们可以用 Python 复刻一下 xTB 初始化速度的过程。

```python
from typing import Union
from rdkit import Chem
import numpy as np


def xTB_initial_velocity(atom_list: Union[list, np.ndarray], t_init: Union[int, float], nvt=True) -> np.ndarray:
    """
    xTB_initial_velocity
    使用 xTB 的方法初始化速度
    :param atom_list: Union[list, np.ndarray]，体系原子序数表
    :param t_init: Union[int, float]，初始温度
    :param nvt=True，NVT系综
    :return: np.ndarray

    Author: Heqi Liu
    GitHub: https://github.com/metaphorme
    Time: 4 April 2024
    Licensed under the MIT License
    """
    
    if isinstance(atom_list, list):
        atom_list = np.array(atom_list)

    # 初始化常数
    amutoau = 1.660539040e-27 / 9.10938356e-31
    kB = 3.166808578545117e-06  # Boltzmann constant in Eh/K

    # 初始化原子质量
    atomic_number_atomic_mass = []
    for i in range(119):
        atomic_number_atomic_mass.append([Chem.Atom.GetMass(Chem.Atom(i)) * amutoau])
    atomic_number_atomic_mass = np.array(atomic_number_atomic_mass)  # 质量（单位：电子质量 Me），列表index索引代表第index个元素，第0个元素为0。

    nfreedom = len(atom_list) * 3  # 自由度
    f = 1 if nvt else 2  # nvt 系综

    edum = f * t_init * 0.5 * kB * nfreedom
    eperat = edum / (3.0 * len(atom_list))

    f = np.random.choice([-1, 1], size=(len(atom_list), 3))  # 均匀分布的 array
    f2 = np.where(atom_list == 1, 2, 1).reshape(-1, 1)  # 如果原子是 H，则 f2 = 2，否则 f2 = 1；并转换为列向量
    f2 = np.repeat(f2, 3, axis=1)  # 横向扩展 3 次

    masses = atomic_number_atomic_mass[atom_list]  # 原子序数对应质量
    masses = np.repeat(masses, 3, axis=1)  # 横向扩展 3 次

    v = np.sqrt(2 * eperat / masses)
    velocity = v * f * f2
    return velocity
```

我们可以尝试使用 xTB 的速度初始化方法初始化 C2H4O2 在 300K 时的速度，NVT 系综：

```python
# 使用 xTB 的速度初始化方法初始化 C2H4O2 在 300K 时的速度，NVT 系综
velocities = xTB_initial_velocities(atom_list=[8, 8, 6, 6, 1, 1, 1, 1], t_init=300, nvt=True)
np.set_printoptions(precision=14)  # xTB 速度显示为 14 位小数
print(velocities)
```

得到的初始速度如下：

```
[[-0.00018048664152 -0.00018048664152 -0.00018048664152]
 [ 0.00018048664152  0.00018048664152 -0.00018048664152]
 [ 0.00020830605754  0.00020830605754 -0.00020830605754]
 [ 0.00020830605754 -0.00020830605754  0.00020830605754]
 [-0.00143810704072 -0.00143810704072  0.00143810704072]
 [ 0.00143810704072 -0.00143810704072  0.00143810704072]
 [-0.00143810704072  0.00143810704072  0.00143810704072]
 [ 0.00143810704072 -0.00143810704072  0.00143810704072]]
```

我们验证一下 xTB 在相同体系下初始化速度结果。

AceticAcid.xyz

```
8
176
O          3.73200        0.75000        0.00000
O          2.86600       -0.75000        0.00000
C          2.00000        0.75000        0.00000
C          2.86600        0.25000        0.00000
H          2.31000        1.28690        0.00000
H          1.46310        1.06000        0.00000
H          1.69000        0.21310        0.00000
H          4.26900        0.44000        0.00000
```

omd.inp

```
$md
   temp= 300  
   time= 1
   dump= 1.0  
   step= 1.0  
   hmass=1 
   shake=0  
   velo=1
```

使用命令

```bash
xtb AceticAcid.xyz --input omd.inp --omd --gfn 2
```

查看 xtb.trj 的第一帧：

```
8
 energy: -14.459893411480 gnorm: 0.001603203349 xtb: 6.6.1 (8d0f1dd)
O            4.09846674214441        0.84974139198898       -0.02892014193760
O            3.10551819666819       -1.14000787634717       -0.16185260087129
C            1.74763035741186        0.83735943412236        0.08073596092055
C            3.02113275499960        0.05176783214537       -0.04500757854942
H            1.92926331427316        1.76005298883354        0.62166358152940
H            1.39242185446835        1.07935383460586       -0.91994986572424
H            0.99124430427879        0.23613563791291        0.57655444828163
H            4.91042247575559        0.32559675673834       -0.12322380364853
   -0.00018048435759   -0.00018048435759   -0.00018048435759
    0.00018048435759   -0.00018048435759    0.00018048435759
    0.00020830834772    0.00020830834772   -0.00020830834772
   -0.00020830834772    0.00020830834772   -0.00020830834772
   -0.00144384800982    0.00144384800982    0.00144384800982
   -0.00144384800982    0.00144384800982    0.00144384800982
    0.00144384800982   -0.00144384800982   -0.00144384800982
   -0.00144384800982    0.00144384800982    0.00144384800982
```

可以看到初始化速度几乎一致。部分小数不同是因为 xTB 自带的原子质量与 rdkit 提供的原子质量有差异，以及运算精度差异。

以上代码 commit 为 [a73499f](https://github.com/grimme-lab/xtb/commit/a73499f9515d9e48614699d70f63fa908a699c38)，初始化方法可能在未来发生变动。

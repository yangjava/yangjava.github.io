---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL源码MySQL Group Replication 的新主选举算法
MGR 的新主选举算法，在节点版本一致的情况下，其实也挺简单的。

首先比较权重，权重越高，选为新主的优先级越高。

如果权重一致，则会进一步比较节点的 server_uuid。server_uuid 越小，选为新主的优先级越高。

所以，在节点版本一致的情况下，会选择权重最高，server_uuid 最小的节点作为新的主节点。

节点的权重由 group_replication_member_weight 决定，该参数是 MySQL 5.7.20 引入的，可设置 0 到 100 之间的任意整数值，默认是 50。

但如果集群节点版本不一致，实际的选举算法就没这么简单了。

下面，我们结合源码具体分析下。

## 代码实现逻辑
新主选举算法主要会涉及三个函数：

pick_primary_member
sort_and_get_lowest_version_member_position
sort_members_for_election
这三个函数都是在 primary_election_invocation_handler.cc 中定义的。

其中，pick_primary_member 是主函数，它会基于其它两个函数的结果选择 Primary 节点。

下面，我们从 pick_primary_member 出发，看看这三个函数的具体实现逻辑。

### pick_primary_member
```
bool Primary_election_handler::pick_primary_member(
    std::string &primary_uuid,
    std::vector<Group_member_info *> *all_members_info) {
  DBUG_TRACE;

  bool am_i_leaving = true;
#ifndef NDEBUG
  int n = 0;
#endif
  Group_member_info *the_primary = nullptr;

  std::vector<Group_member_info *>::iterator it;
  std::vector<Group_member_info *>::iterator lowest_version_end;

  // 基于 member_version 选择候选节点。
  lowest_version_end =
      sort_and_get_lowest_version_member_position(all_members_info);

  // 基于节点权重和 server_uuid 对候选节点进行排序。
  sort_members_for_election(all_members_info, lowest_version_end);


  // 遍历所有节点，判断 Primary 节点是否已定义。
  for (it = all_members_info->begin(); it != all_members_info->end(); it++) {
#ifndef NDEBUG
    assert(n <= 1);
#endif

    Group_member_info *member = *it;
    // 如果当前节点是单主模式且遍历的节点中有 Primary 节点，则将该节点赋值给 the_primary
    if (local_member_info->in_primary_mode() && the_primary == nullptr &&
        member->get_role() == Group_member_info::MEMBER_ROLE_PRIMARY) {
      the_primary = member;
#ifndef NDEBUG
      n++;
#endif
    }

    // 检查当前节点的状态是否为 OFFLINE。
    if (!member->get_uuid().compare(local_member_info->get_uuid())) {
      am_i_leaving =
          member->get_recovery_status() == Group_member_info::MEMBER_OFFLINE;
    }
  }

  // 如果当前节点的状态不是 OFFLINE 且 the_primary 还是为空，则选择一个 Primary 节点
  if (!am_i_leaving) {
    if (the_primary == nullptr) {
      // 因为循环的结束条件是 it != lowest_version_end 且 the_primary 为空，所以基本上会将候选节点中的第一个节点作为 Primary 节点。
      for (it = all_members_info->begin();
           it != lowest_version_end && the_primary == nullptr; it++) {
        Group_member_info *member_info = *it;

        assert(member_info);
        if (member_info && member_info->get_recovery_status() ==
                               Group_member_info::MEMBER_ONLINE)
          the_primary = member_info;
      }
    }
  }

  if (the_primary == nullptr) return true;

  primary_uuid.assign(the_primary->get_uuid());
  return false;
}
```
这个函数里面，比较关键的地方有三个：

调用 sort_and_get_lowest_version_member_position。

这个函数会基于 member_version （节点版本）选择候选节点。

只有候选节点才有资格被选为主节点 。

调用 sort_members_for_election。

这个函数会基于节点权重和 server_uuid，对候选节点进行排序。

基于排序后的候选节点选择 Primary 节点。

因为候选节点是从头开始遍历，所以基本上，只要第一个节点是 ONLINE 状态，就会把这个节点作为 Primary 节点。

### sort_and_get_lowest_version_member_position
接下来我们看看 sort_and_get_lowest_version_member_position 函数的实现逻辑。
```
sort_and_get_lowest_version_member_position(
    std::vector<Group_member_info *> *all_members_info) {
  std::vector<Group_member_info *>::iterator it;

  // 按照版本对 all_members_info 从小到大排序
  std::sort(all_members_info->begin(), all_members_info->end(),
            Group_member_info::comparator_group_member_version);

  // std::vector::end 会返回一个迭代器，该迭代器引用 vector （向量容器）中的末尾元素。
  // 注意，这个元素指向的是 vector 最后一个元素的下一个位置，不是最后一个元素。
  std::vector<Group_member_info *>::iterator lowest_version_end =
      all_members_info->end();

  // 获取排序后的第一个节点，这个节点版本最低。
  it = all_members_info->begin();
  Group_member_info *first_member = *it;
  // 获取第一个节点的 major_version
  // 对于 MySQL 5.7，major_version 是 5；对于 MySQL 8.0，major_version 是 8
  uint32 lowest_major_version =
      first_member->get_member_version().get_major_version();
  
  /* to avoid read compatibility issue leader should be picked only from lowest
     version members so save position where member version differs.
     From 8.0.17 patch version will be considered during version comparison.

     set lowest_version_end when major version changes

     eg: for a list: 5.7.18, 5.7.18, 5.7.19, 5.7.20, 5.7.21, 8.0.2
         the members to be considered for election will be:
            5.7.18, 5.7.18, 5.7.19, 5.7.20, 5.7.21
         and server_uuid based algorithm will be used to elect primary

     eg: for a list: 5.7.20, 5.7.21, 8.0.2, 8.0.2
         the members to be considered for election will be:
            5.7.20, 5.7.21
         and member weight based algorithm will be used to elect primary

     eg: for a list: 8.0.17, 8.0.18, 8.0.19
         the members to be considered for election will be:
            8.0.17

     eg: for a list: 8.0.13, 8.0.17, 8.0.18
         the members to be considered for election will be:
            8.0.13, 8.0.17, 8.0.18
         and member weight based algorithm will be used to elect primary
  */
  

  // 遍历剩下的节点，注意 it 是从 all_members_info->begin() + 1 开始的
  for (it = all_members_info->begin() + 1; it != all_members_info->end();
       it++) {
    // 如果第一个节点的版本号大于 MySQL 8.0.17，且节点的版本号不等于第一个节点的版本号，则将该节点赋值给 lowest_version_end，并退出循环。
    if (first_member->get_member_version() >=
            PRIMARY_ELECTION_PATCH_CONSIDERATION &&
        (first_member->get_member_version() != (*it)->get_member_version())) {
      lowest_version_end = it;
      break;
    }
    // 如果节点的 major_version 不等于第一个节点的 major_version，则将该节点赋值给 lowest_version_end，并退出循环。
    if (lowest_major_version !=
        (*it)->get_member_version().get_major_version()) {
      lowest_version_end = it;
      break;
    }
  }
  return lowest_version_end;
}
```
函数中的 PRIMARY_ELECTION_PATCH_CONSIDERATION 是 0x080017，即 MySQL 8.0.17。

在 MySQL 8.0.17 中，Group Replication 引入了兼容性策略。引入兼容性策略的初衷是为了避免集群中出现节点不兼容的情况。

该函数首先会对 all_members_info 按照版本从小到大排序。

接着会基于第一个节点的版本（最小版本）确定 lowest_version_end。

MGR 用 lowest_version_end 标记最低版本的结束点。只有 lowest_version_end 之前的节点才是候选节点。

lowest_version_end 的取值逻辑如下：

如果最小版本大于等于 MySQL 8.0.17，则会将最小版本之后的第一个节点设置为 lowest_version_end。
如果集群中既有 5.7，又有 8.0，则会将 8.0 的第一个节点设置为 lowest_version_end。
如果最小版本小于 MySQL 8.0.17，且只有一个大版本（major_version），则会取 all_members_info->end()。此时，所有节点都是候选节点。
为了方便大家理解代码的逻辑，函数注释部分还列举了四个案例，每个案例对应一个典型场景。后面我们会具体分析下。

## sort_members_for_election
最后，我们看看 sort_members_for_election 函数的实现逻辑。
```
void sort_members_for_election(
    std::vector<Group_member_info *> *all_members_info,
    std::vector<Group_member_info *>::iterator lowest_version_end) {
  Group_member_info *first_member = *(all_members_info->begin());
  // 获取第一个节点的版本，这个节点版本最低。
  Member_version lowest_version = first_member->get_member_version();

  // 如果最小版本大于等于 MySQL 5.7.20，则根据节点的权重来排序。权重越高，在 vector 中的位置越靠前。
  // 注意，这里只会对 [all_members_info->begin(), lowest_version_end) 这个区间内的元素进行排序，不包括 lowest_version_end。
  if (lowest_version >= PRIMARY_ELECTION_MEMBER_WEIGHT_VERSION)
    std::sort(all_members_info->begin(), lowest_version_end,
              Group_member_info::comparator_group_member_weight);
  else
    // 如果最小版本小于 MySQL 5.7.20，则根据节点的 server_uuid 来排序。server_uuid 越小，在 vector 中的位置越靠前。
    std::sort(all_members_info->begin(), lowest_version_end,
              Group_member_info::comparator_group_member_uuid);
}
```
函数中的 PRIMARY_ELECTION_MEMBER_WEIGHT_VERSION 是 0x050720，即 MySQL 5.7.20。

如果最小节点的版本大于等于 MySQL 5.7.20，则会基于权重来排序。权重越高，在 all_members_info 中的位置越靠前。

如果最小节点的版本小于 MySQL 5.7.20，则会基于节点的 server_uuid 来排序。server_uuid 越小，在 all_members_info 中的位置越靠前。

注意，std::sort 中的结束位置是 lowest_version_end，所以 lowest_version_end 这个节点不会参与排序。

## comparator_group_member_weight
在基于权重进行排序时，如果两个节点的权重一致，还会进一步比较这两个节点的 server_uuid。

这个逻辑是在 comparator_group_member_weight 中定义的。

权重一致，节点的 server_uuid 越小，在 all_members_info 中的位置越靠前。
```
bool Group_member_info::comparator_group_member_weight(Group_member_info *m1,
                                                       Group_member_info *m2) {
  return m1->has_greater_weight(m2);
}

bool Group_member_info::has_greater_weight(Group_member_info *other) {
  MUTEX_LOCK(lock, &update_lock);
  if (member_weight > other->get_member_weight()) return true;
  // 如果权重一致，会按照节点的 server_uuid 来排序。
  if (member_weight == other->get_member_weight())
    return has_lower_uuid_internal(other);

  return false;
}
```
案例分析
基于上面代码的逻辑，接下来我们分析下 sort_and_get_lowest_version_member_position 函数注释部分列举的四个案例：

案例 1：5.7.18, 5.7.18, 5.7.19, 5.7.20, 5.7.21, 8.0.2
这几个节点中，最小版本号是 5.7.18，小于 MySQL 8.0.17。所以会比较各个节点的 major_version，因为最后一个节点（8.0.2）的 major_version 和第一个节点不一致，所以会将 8.0.2 作为 lowest_version_end。此时，除了 8.0.2，其它都是候选节点。

最小版本号 5.7.18 小于 MySQL 5.7.20，所以 5.7.18, 5.7.18, 5.7.19, 5.7.20, 5.7.21 这几个节点会根据 server_uuid 进行排序。注意，lowest_version_end 的节点不会参与排序。

选择 server_uuid 最小的节点作为 Primary 节点。

案例 2：5.7.20, 5.7.21, 8.0.2, 8.0.2
同案例 1 一样，会将 8.0.2 作为 lowest_version_end。此时，候选节点只有 5.7.20 和 5.7.21。

最小版本号 5.7.20 等于 MySQL 5.7.20，所以，5.7.20, 5.7.21 这两个节点会根据节点的权重进行排序。如果权重一致，则会基于 server_uuid 进行进一步的排序。

选择权重最高，server_uuid 最小的节点作为 Primary 节点。

案例 3：8.0.17, 8.0.18, 8.0.19
最小版本号是 MySQL 8.0.17，等于 MySQL 8.0.17，所以会判断其它节点的版本号是否与第一个节点相同。不相同，则会将该节点的版本号赋值给 lowest_version_end。所以，会将 8.0.18 作为 lowest_version_end。此时，候选节点只有 8.0.17。

选择 8.0.17 这个节点作为 Primary 节点。

案例 4：8.0.13, 8.0.17, 8.0.18
最小版本号是 MySQL 8.0.13，小于 MySQL 8.0.17，而且各个节点的 major_version 一致，所以最后返回的 lowest_version_end 实际上是 all_members_info->end()。此时，这三个节点都是候选节点。

MySQL 8.0.13 大于 MySQL 5.7.20，所以这三个节点会根据权重进行排序。如果权重一致，则会基于 server_uuid 进行进一步的排序。

选择权重最高，server_uuid 最小的节点作为 Primary 节点。

## 手动选主
从 MySQL 8.0.13 开始，我们可以通过以下两个函数手动选择新的主节点：

group_replication_set_as_primary(server_uuid) ：切换单主模式下的 Primary 节点。
group_replication_switch_to_single_primary_mode([server_uuid]) ：将多主模式切换为单主模式。可通过 server_uuid 指定单主模式下的 Primary 节点。
在使用这两个参数时，注意，指定的 server_uuid 必须属于候选节点。

另外，这两个函数是 MySQL 8.0.13 引入的，所以，如果集群中存在 MySQL 8.0.13 之前的节点，执行时会报错。
```
mysql> select group_replication_set_as_primary('5470a304-3bfa-11ed-8bee-83f233272a5d');
ERROR 3910 (HY000): The function 'group_replication_set_as_primary' failed. The group has a member with a version that does not support group coordinated operations.
```
总结
结合代码和上面四个案例的分析，最后我们总结下 MGR 的新主选举算法：

如果集群中存在 MySQL 5.7 的节点，则会将 MySQL 5.7 的节点作为候选节点。

如果集群节点的版本都是 MySQL 8.0，这里需要区分两种情况：

如果最小版本小于 MySQL 8.0.17，则所有的节点都可作为候选节点。
如果最小版本大于等于 MySQL 8.0.17，则只有最小版本的节点会作为候选节点。
在候选节点的基础上，会进一步根据候选节点的权重和 server_uuid 选择 Primary 节点。具体来说，

如果候选节点中存在 MySQL 5.7.20 之前版本的节点，则会选择 server_uuid 最小的节点作为 Primary 节点。
如果候选节点都大于等于 MySQL 5.7.20，则会选择权重最高，server_uuid 最小的节点作为 Primary 节点。

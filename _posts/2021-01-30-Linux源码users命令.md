---
layout: post
categories: [Linux]
description: none
keywords: Linux
---
# Linux源码users命令

## 命令功能：显示当前所有的登陆用户
users的操作数是指定文件获取

通过utmp(/var/run/utmp)获取当前登陆用户，utmp不存在就使用wtmp(/var/log/wtmp)用户操作记录文件来获取

如果多次同一用户名，就是这个用户同时存在多个会话

核心函数：read_utmp定义在utmp.h 读取utmp文件

执行顺序：判断是否有指定utmp文件，没的话就使用默认的/var/run/utmp，然后使用read_utmp读取utmp文件，然后在遍历读取到的用户
```
/* GNU's users.
   Copyright (C) 1992-2020 Free Software Foundation, Inc.
   This program is free software: you can redistribute it and/or modify
   it under the terms of the GNU General Public License as published by
   the Free Software Foundation, either version 3 of the License, or
   (at your option) any later version.
   This program is distributed in the hope that it will be useful,
   but WITHOUT ANY WARRANTY; without even the implied warranty of
   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
   GNU General Public License for more details.
   You should have received a copy of the GNU General Public License
   along with this program.  If not, see <https://www.gnu.org/licenses/>.  */
 
/* Written by jla; revised by djm */
 
#include <config.h>
#include <stdio.h>
 
#include <sys/types.h>
#include "system.h"
 
#include "die.h"
#include "error.h"
#include "long-options.h"
#include "quote.h"
#include "readutmp.h"
 
/* 程序名 */
#define PROGRAM_NAME "users"
 
/* 作者名 */
#define AUTHORS \
  proper_name ("Joseph Arceneaux"), \
  proper_name ("David MacKenzie")
 
static int
userid_compare (const void *v_a, const void *v_b)
{
  char **a = (char **) v_a;
  char **b = (char **) v_b;
  return strcmp (*a, *b);
}
 
/* 遍历输出utmp列表的用户 */
static void
list_entries_users (size_t n, const STRUCT_UTMP *this)
{
  char **u = xnmalloc (n, sizeof *u);
  size_t i;
  size_t n_entries = 0;
 
  while (n--)
    {
      if (IS_USER_PROCESS (this))
        {
          char *trimmed_name;
 
          trimmed_name = extract_trimmed_name (this);
 
          u[n_entries] = trimmed_name;
          ++n_entries;
        }
      this++;
    }
 
  qsort (u, n_entries, sizeof (u[0]), userid_compare);
 
  for (i = 0; i < n_entries; i++)
    {
      char c = (i < n_entries - 1 ? ' ' : '\n');
      fputs (u[i], stdout);
      putchar (c);
    }
 
  for (i = 0; i < n_entries; i++)
    free (u[i]);
  free (u);
}
 
/* 根据utmp文件FILENAME显示系统上的用户列表。
使用read_utmp选项来读取文件名。  */
static void
users (const char *filename, int options)
{
  size_t n_users;
  STRUCT_UTMP *utmp_buf; /* utmp结构 */
 
  /* 读取utmp文件 */
  if (read_utmp (filename, &n_users, &utmp_buf, options) != 0)
    die (EXIT_FAILURE, errno, "%s", quotef (filename));
 
  /* 遍历输出utmp_buf列表的用户名 */
  list_entries_users (n_users, utmp_buf);
 
  free (utmp_buf);
}
 
/* 用法函数 */
void
usage (int status)
{
  if (status != EXIT_SUCCESS)
    emit_try_help ();
  else
    {
      printf (_("Usage: %s [OPTION]... [FILE]\n"), program_name);
      printf (_("\
Output who is currently logged in according to FILE.\n\
If FILE is not specified, use %s.  %s as FILE is common.\n\
\n\
"),
              UTMP_FILE, WTMP_FILE);
      fputs (HELP_OPTION_DESCRIPTION, stdout);
      fputs (VERSION_OPTION_DESCRIPTION, stdout);
      emit_ancillary_info (PROGRAM_NAME);
    }
  exit (status);
}
 
int
main (int argc, char **argv)
{
  initialize_main (&argc, &argv);
  set_program_name (argv[0]);
  setlocale (LC_ALL, "");
  bindtextdomain (PACKAGE, LOCALEDIR);
  textdomain (PACKAGE);
 
  atexit (close_stdout);
 
  parse_gnu_standard_options_only (argc, argv, PROGRAM_NAME, PACKAGE_NAME,
                                   Version, true, usage, AUTHORS,
                                   (char const *) NULL);
 
  /* 默认：users   argc=1   optind=1
   * 带指定文件：users /var/log/utmp */
  switch (argc - optind)
    {
    case 0:			/* 默认 */
      users (UTMP_FILE, READ_UTMP_CHECK_PIDS);
      break;
 
    case 1:			/* 指定用户(utmp)文件 */
      users (argv[optind], 0);
      break;
 
    default:			/* lose */
      error (0, 0, _("extra operand %s"), quote (argv[optind + 1]));
      usage (EXIT_FAILURE);
    }
 
  return EXIT_SUCCESS;
}
```
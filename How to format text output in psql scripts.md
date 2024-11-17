# How to format text output in psql scripts

对于 `psql` 的 `\echo` 命令，可以通过使用 ANSI 颜色代码来实现颜色和基本的文本格式化，比如加粗或下划线。这在构建复杂的 psql 脚本时非常有用 (比如 [postgres_dba](https://github.com/NikolayS/postgres_dba/))。

例如：

```bash
\echo '\033[1;31mThis is red text\033[0m'
\echo '\033[1;32mThis is green text\033[0m'
\echo '\033[1;33mThis is yellow text\033[0m'
\echo '\033[1;34mThis is blue text\033[0m'
\echo '\033[1;35mThis is magenta text\033[0m'
\echo '\033[1;36mThis is cyan text\033[0m'
\echo '\033[1mThis text is bold\033[0m'
\echo '\033[4mThis text is underlined\033[0m'
\echo '\033[38;2;128;0;128mThis text is purple\033[0m'

-- RGB – arbitrary color
\echo '\033[38;2;255;100;0mThis text is in orange (RGB 255,100,0)\033[0m'
```

结果：

![img](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0091_result.png)

**重要提示：**`\033[0m` 序列会将文本格式重置为默认格式。 

在 `psql` 的非交互模式中，格式会被保留，且与 `ts` 命令结合使用时也会保留 (前缀时间戳，`ts` 是 Ubuntu 中 moreutils 包的一部分)：

```bash
psql -Xc "\echo '\033[1;35m这是品红色文本\033[0m'" | ts
```

当使用 `less` 命令时，格式不会生效，但可以通过 `-R` 选项来解决：

```bash
psql -Xc "\echo '\033[1;35m这是品红色文本\033[0m'" | less -R
```
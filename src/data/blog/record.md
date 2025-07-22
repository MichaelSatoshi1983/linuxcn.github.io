---
title: "回响：灯塔长明 航迹不忘"
date: "2025-07-16 12:37:42"
pubDatetime: 2025-07-16 12:37:42
description: "介绍了通过 Astro 搭建镜像站的方法和从Hexo迁移到Astro的一些思路"
tags: ["Linux 中国"]
---
# 搭建过程
感谢 Linux CN 团队及其相关成员留下了如此宝贵的财富。
## 通过 Astro 搭建（从原Hexo迁移）
直接将markdown文章拷贝到主题的文章目录下即可。
## 自动把文章格式化为 Astro 支持的样式。
原本的文章的 yaml 数据和 Astro 的不通用，这里就使用 Python 写了个小脚本：
> 这个脚本只保留了 `title`、`date`、`pubDatetime`、`description` 和 `tags` 哦，有需要其他标签可以自己增加修改。
```python
import os
import re
import yaml
import sys
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime


def convert_yaml(old_meta):
    """转换YAML元数据到新格式"""
    new_meta = {}

    # 处理title
    new_meta['title'] = old_meta.get('title', '')

    # 处理date和pubDatetime（优先使用updated字段）
    date_value = old_meta.get('updated', old_meta.get('date', ''))
    # 确保去除可能存在的引号
    if isinstance(date_value, str):
        date_value = date_value.strip('"').strip("'")
    new_meta['date'] = f'"{date_value}"'  # 带引号的字符串
    new_meta['pubDatetime'] = date_value  # 不带引号

    # 处理description（使用summary字段）
    description = old_meta.get('summary', '')
    # 转义双引号防止YAML解析错误
    new_meta['description'] = description.replace('"', '\\"')

    # 处理tags
    old_tags = old_meta.get('tags', [])
    if isinstance(old_tags, str):
        new_meta['tags'] = [old_tags]
    elif isinstance(old_tags, list):
        new_meta['tags'] = [str(tag) for tag in old_tags]
    else:
        new_meta['tags'] = []

    # 按指定顺序返回字段
    return {k: new_meta[k] for k in ['title', 'date', 'pubDatetime', 'description', 'tags']}


def process_file(file_path):
    """处理单个Markdown文件"""
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()

        # 匹配YAML front matter
        match = re.match(r'^---\s*\n(.+?)\n---\s*\n(.*)$', content, re.DOTALL)
        if not match:
            print(f"⚠️  {file_path}: 未找到YAML front matter，跳过")
            return

        yaml_content, markdown_content = match.groups()

        # 解析旧YAML
        old_meta = yaml.safe_load(yaml_content)
        print(f"🔧 处理中: {file_path}")
        print(f"   原始元数据: {list(old_meta.keys())}")

        # 转换元数据
        new_meta = convert_yaml(old_meta)

        # 构建新YAML
        new_yaml = "---\n"
        new_yaml += f"title: \"{new_meta['title']}\"\n"
        new_yaml += f"date: {new_meta['date']}\n"
        # 关键修复：pubDatetime不带引号
        new_yaml += f"pubDatetime: {new_meta['pubDatetime']}\n"
        new_yaml += f"description: \"{new_meta['description']}\"\n"

        # 处理tags数组
        tags_str = "[" + ", ".join(f'"{tag}"' for tag in new_meta['tags']) + "]"
        new_yaml += f"tags: {tags_str}\n"
        new_yaml += "---\n\n"

        # 写入新内容
        with open(file_path, 'w', encoding='utf-8') as f:
            f.write(new_yaml + markdown_content)

        print(f"✅ 完成: {file_path}")
        print(f"   新元数据: pubDatetime={new_meta['pubDatetime']} (不带引号)")

    except Exception as e:
        print(f"❌ 处理失败: {file_path} - {str(e)}")
        import traceback
        traceback.print_exc()


def main(directory):
    """主函数，处理目录中的所有Markdown文件"""
    start_time = datetime.now()
    file_count = 0

    # 收集所有Markdown文件
    files = []
    for root, _, filenames in os.walk(directory):
        for filename in filenames:
            if filename.endswith('.md'):
                files.append(os.path.join(root, filename))

    print(f"📁 找到 {len(files)} 个Markdown文件")

    # 使用线程池处理文件
    with ThreadPoolExecutor(max_workers=os.cpu_count() * 4) as executor:
        futures = [executor.submit(process_file, f) for f in files]

        for future in as_completed(futures):
            try:
                future.result()  # 触发异常捕获
                file_count += 1
            except Exception as e:
                print(f"❌ 线程处理异常: {str(e)}")

    elapsed = (datetime.now() - start_time).total_seconds()
    print(f"\n🎉 所有文件处理完成! 耗时: {elapsed:.2f}秒, 处理文件数: {file_count}/{len(files)}")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("使用方法: python3 convert_yaml.py <目录>")
        sys.exit(1)

    target_dir = sys.argv[1]
    if not os.path.isdir(target_dir):
        print(f"错误: {target_dir} 不是有效目录")
        sys.exit(1)

    main(target_dir)
```
## 自动生成文章摘要和摘要的优化
### 自动生成摘要
自动生成摘要的话可以去我的博客那[看一下](https://undefined.today/posts/auto-description/)
### 摘要优化
默认生成的摘要可能会出现很多意外的符号，例如双引号没做转义啊之类的。我这里同样写了个脚本（只处理了双引号问题）：
```python
import os
import re
import sys
import threading
from concurrent.futures import ThreadPoolExecutor

# 匹配description行的正则表达式
pattern = re.compile(r'^(\s*description:\s*)"(.*)"\s*$', re.DOTALL)

# 存储需要修改的文件信息 {文件路径: [(行号, 原行内容, 新行内容)]}
file_changes = {}
lock = threading.Lock()

# ANSI颜色代码
GREEN = '\033[92m'
RED = '\033[91m'
RESET = '\033[0m'


def process_file(file_path):
    """处理单个文件，识别需要修改的行"""
    changes = []
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()

        in_front_matter = False
        front_matter_end = False

        for idx, line in enumerate(lines):
            stripped_line = line.strip()

            # 检测front matter边界
            if stripped_line == '---':
                if not in_front_matter:
                    in_front_matter = True
                else:
                    front_matter_end = True
                continue

            # 只在front matter内处理
            if in_front_matter and not front_matter_end:
                match = pattern.match(line)
                if match:
                    prefix = match.group(1)
                    content = match.group(2)

                    # 统计双引号数量
                    quote_count = content.count('"')

                    # 只有多余的双引号才处理
                    if quote_count > 0:
                        # 直接去掉所有内容中的双引号
                        cleaned_content = content.replace('"', '')
                        new_line = f'{prefix}"{cleaned_content}"\n'

                        # 保留原始缩进
                        if not line.endswith('\n'):
                            new_line = new_line.rstrip('\n')

                        changes.append((idx, line, new_line))

        if changes:
            with lock:
                file_changes[file_path] = changes

    except Exception as e:
        print(f"{RED}处理 {file_path} 时出错: {str(e)}{RESET}")


def main():
    if len(sys.argv) != 2:
        print(f"{RED}使用方法: python3 fix_description.py <目录路径>{RESET}")
        sys.exit(1)

    root_dir = sys.argv[1]
    md_files = []
    unmodified_files = []

    # 收集所有Markdown文件
    for root, _, files in os.walk(root_dir):
        for file in files:
            if file.endswith('.md'):
                md_files.append(os.path.join(root, file))

    if not md_files:
        print(f"{GREEN}未找到Markdown文件{RESET}")
        return

    print(f"扫描目录: {root_dir}")
    print(f"找到 {len(md_files)} 个Markdown文件")

    # 使用多线程处理文件
    with ThreadPoolExecutor() as executor:
        executor.map(process_file, md_files)

    # 收集未修改的文件
    for file in md_files:
        if file not in file_changes:
            unmodified_files.append(file)

    # 显示未修改的文件（绿色）
    if unmodified_files:
        print(f"\n{GREEN}以下文件无需修改:{RESET}")
        for file in unmodified_files:
            print(f"  {GREEN}{file}{RESET}")

    # 显示需要修改的文件（红色）
    if file_changes:
        print(f"\n{RED}以下文件需要修改:{RESET}")
        for file_path, changes in file_changes.items():
            print(f"\n{RED}文件: {file_path}{RESET}")
            for idx, (line_num, old_line, new_line) in enumerate(changes):
                print(f"修改 {idx + 1}:")
                print(f"  {RED}原行: {old_line.strip()}{RESET}")
                print(f"  {GREEN}新行: {new_line.strip()}{RESET}")

        # 等待用户确认
        confirm = input("\n确认修改? (y/n): ").lower()
        if confirm != 'y':
            print(f"{GREEN}操作取消{RESET}")
            return

        # 执行修改
        modified_count = 0
        for file_path, changes in file_changes.items():
            with open(file_path, 'r', encoding='utf-8') as f:
                lines = f.readlines()

            for line_num, _, new_line in changes:
                if line_num < len(lines):
                    lines[line_num] = new_line

            with open(file_path, 'w', encoding='utf-8') as f:
                f.writelines(lines)
            modified_count += 1

        print(f"\n{GREEN}成功修改 {modified_count} 个文件{RESET}")
    else:
        print(f"\n{GREEN}所有文件均无需修改{RESET}")


if __name__ == "__main__":
    main()
```
直接使用即可。
## 处理文章内的图片
文章内包含了大量的图片需要去处理，为此我编写了一个[脚本](https://github.com/MichaelSatoshi1983/archive)，下载后需要只需要执行的时候跟上路径和要替换的cdn地址即可，他会自己分析。

当然这里也可以直接使用：
```python
import os
import re
import argparse
import concurrent.futures
import time


def process_file(filepath, cdn_base_url):
    """处理单个Markdown文件"""
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()

    # 编译正则表达式：匹配Markdown中的图片路径
    pattern = re.compile(
        r'('
        r'(!\[[^\]]*\]\()'  # 标准Markdown图片语法
        r'|'  # 或
        r'(\n\s*[a-zA-Z0-9_-]+\s*:)'  # YAML front matter键值对
        r')'
        r'(/data/(?:attachment/album|img)/[^\)\s]+)'  # 图片路径
    )

    replacements = []
    new_content = content

    # 查找所有匹配项
    matches = list(pattern.finditer(content))
    if not matches:
        return 0

    # 构建替换列表
    for match in matches:
        old_path = match.group(4)
        new_path = f"{cdn_base_url}{old_path}"
        replacements.append((old_path, new_path))

    # 一次性替换所有匹配项（反向替换避免索引偏移）
    for old_path, new_path in reversed(replacements):
        new_content = new_content.replace(old_path, new_path)

    # 写回文件
    with open(filepath, 'w', encoding='utf-8') as f:
        f.write(new_content)

    return len(replacements)


def replace_md_image_paths(directory, cdn_base_url, workers=8):
    """
    递归遍历目录，替换Markdown文件中的本地图片路径为CDN路径
    :param directory: 要遍历的目录路径
    :param cdn_base_url: CDN基础URL
    :param workers: 并行处理线程数
    """
    cdn_base_url = cdn_base_url.rstrip('/')
    file_queue = []
    total_files = 0
    total_replacements = 0

    # 收集所有需要处理的文件
    for root, _, files in os.walk(directory):
        for filename in files:
            if filename.lower().endswith('.md'):
                filepath = os.path.join(root, filename)
                file_queue.append(filepath)
                total_files += 1

    print(f"k开始处理: 共发现 {total_files} 个Markdown文件")
    start_time = time.time()

    # 使用线程池并行处理
    with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as executor:
        futures = {executor.submit(process_file, fp, cdn_base_url): fp for fp in file_queue}

        for i, future in enumerate(concurrent.futures.as_completed(futures), 1):
            filepath = futures[future]
            try:
                replacements = future.result()
                if replacements > 0:
                    print(f"处理进度: {i}/{total_files} | 文件: {filepath} | 替换: {replacements}处")
                    total_replacements += replacements
            except Exception as e:
                print(f"\033[31m处理失败: {filepath} | 错误: {str(e)}\033[0m")

    elapsed = time.time() - start_time
    print(f"\n\033[1;32m处理完成! 共处理 {total_files} 个文件, 执行 {total_replacements} 次替换")
    print(f"总耗时: {elapsed:.2f}秒 | 平均速度: {total_files / elapsed:.1f} 文件/秒\033[0m")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='替换Markdown中的本地图片路径为CDN路径')
    parser.add_argument('directory', help='包含Markdown文件的根目录')
    parser.add_argument('cdn_url', help='CDN基础URL（如https://cdn.example.com）')
    parser.add_argument('--workers', type=int, default=8, help='并行处理线程数（默认是8）')
    args = parser.parse_args()

    replace_md_image_paths(args.directory, args.cdn_url, args.workers)

```
### 图片资源防刷
针对图片防刷问题，我使用的方法是直接缓存一年。
---
当海面收起最后的帆影
光在塔顶成为凝望
那守望者隐入薄雾
却让光在风里长久流转

于是每个夜晚的波纹
都举起透明的舵盘
浪花低语着旧航线
将方向刻入更深的蔚蓝

我们俯身拾起散落的星群
如同拾起遗落的灯语
每粒沙都记得光的形状
每个港湾都含住未说的潮声

直到所有告别汇成港口
所有姓名化作礁石上的盐霜
你静立在涛声的折痕里
用长明偿还所有远航的遗忘

不必难过
它只是把火种 撒进了千万个终端
就像当年 它把第一颗星
别进中国开源的衣襟

谨以此文，致敬那个永不褪色的时代注脚。愿开源的火种，在数字荒原上永远燃烧。
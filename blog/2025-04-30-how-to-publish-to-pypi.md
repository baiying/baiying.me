要将 Python 开发的命令行工具发布出去供他人使用，可按以下步骤操作：

### 1. 项目结构与配置
- **项目结构**：构建一个合理的项目结构，示例如下：
```plaintext
my_command_line_tool/
├── my_command_line_tool/
│   ├── __init__.py
│   └── main.py
├── tests/
│   └── test_main.py
├── setup.py 或 pyproject.toml
├── README.md
└── LICENSE
```
- **`setup.py` 或 `pyproject.toml`**：这两个文件用于配置项目元数据与依赖项。
    - **`setup.py`**：传统的项目配置文件，示例如下：
```python
from setuptools import setup, find_packages

setup(
    name='my_command_line_tool',
    version='0.1.0',
    packages=find_packages(),
    install_requires=[
        'click',  # 假设使用 click 库创建命令行接口
    ],
    entry_points={
        'console_scripts': [
            'my-cli = my_command_line_tool.main:main',
        ],
    },
)
```
    - **`pyproject.toml`**：新兴的项目配置文件，示例如下：
```toml
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my_command_line_tool"
version = "0.1.0"
description = "A simple command line tool"
authors = [
    { name="Your Name", email="your_email@example.com" },
]
dependencies = [
    "click",
]

[project.scripts]
my-cli = "my_command_line_tool.main:main"
```

### 2. 编写命令行工具代码
以 `click` 库为例，在 `main.py` 中编写命令行工具代码：
```python
import click

@click.command()
@click.option('--name', default='World', help='Your name')
def main(name):
    click.echo(f'Hello, {name}!')

if __name__ == '__main__':
    main()

```

### 3. 测试代码
在 `tests` 目录下编写测试代码，确保工具功能正常。例如，使用 `pytest` 进行测试：
```python
from my_command_line_tool.main import main

def test_main():
    result = main.callback(name='Test')
    assert result is None

```

### 4. 打包项目
- **使用 `setuptools` 打包**：若使用 `setup.py`，在项目根目录下执行以下命令：
```bash
python setup.py sdist bdist_wheel
```
- **使用 `build` 工具打包**：若使用 `pyproject.toml`，先安装 `build` 工具：
```bash
pip install build
```
然后执行以下命令进行打包：
```bash
python -m build
```
打包完成后，会在 `dist` 目录下生成 `.tar.gz` 和 `.whl` 文件。

### 5. 注册并发布到 PyPI
- **注册 PyPI 账号**：访问 [PyPI 官网](https://pypi.org/) 注册账号。
- **安装 `twine`**：用于将包上传到 PyPI：
```bash
pip install twine
```
- **上传包**：在项目根目录下执行以下命令：
```bash
twine upload dist/*
```
按照提示输入 PyPI 账号的用户名和密码，上传成功后，其他人就可以通过 `pip` 安装你的命令行工具：
```bash
pip install my_command_line_tool
```

### 6. 其他发布方式
- **GitHub 发布**：将项目代码上传到 GitHub，用户可以通过 `pip` 直接从 GitHub 安装：
```bash
pip install git+https://github.com/your_username/my_command_line_tool.git
```
- **容器化**：使用 Docker 将命令行工具打包成容器镜像，用户可以通过 Docker 运行工具。

通过以上步骤，你就可以将 Python 开发的命令行工具发布出去供他人使用。 
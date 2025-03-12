# Pre-commit

```yaml
fail_fast: false # 失败后继续运行
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.3.0
    hooks:
      - id: fix-byte-order-marker # 删除UTF-8 字节顺序标记
      - id: check-case-conflict # 检查名称与不区分大小写的文件系统（如 MacOS HFS+ 或 Windows FAT）冲突的文件。
      - id: check-merge-conflict # 检查包含合并冲突字符串的文件。
      - id: check-symlinks # 检查未指向任何内容的符号链接。
      - id: check-yaml # 尝试加载所有 yaml 文件来验证语法。
      - id: end-of-file-fixer # 确保文件以换行符结尾，且仅以换行符结尾。
      - id: mixed-line-ending # 替换或检查混合行结尾。
      - id: trailing-whitespace # 修剪尾随空格。
  - repo: https://github.com/psf/black
    rev: 22.10.0
    hooks:
      - id: black
  - repo: local
    hooks:
      - id: cargo-fmt
        name: cargo fmt
        description: Format files with rustfmt.
        entry: bash -c 'cargo fmt -- --check'
        language: rust
        files: \.rs$
        args: []
      - id: cargo-deny
        name: cargo deny check
        description: Check cargo dependencies
        entry: bash -c 'cargo deny check -d'
        language: rust
        files: \.rs$
        args: []
      - id: typos
        name: typos
        description: check typo
        entry: bash -c 'typos'
        language: rust
        files: \.*$
        pass_filenames: false
      - id: cargo-check
        name: cargo check
        description: Check the package for errors.
        entry: bash -c 'cargo check --all'
        language: rust
        files: \.rs$
        pass_filenames: false
      - id: cargo-clippy
        name: cargo clippy
        description: Lint rust sources
        entry: bash -c 'cargo clippy --all-targets --all-features --tests --benches -- -D warnings'
        language: rust
        files: \.rs$
        pass_filenames: false
      - id: cargo-test
        name: cargo test
        description: unit test for the project
        entry: bash -c 'cargo nextest run --all-features'
        language: rust
        files: \.rs$
        pass_filenames: false
```


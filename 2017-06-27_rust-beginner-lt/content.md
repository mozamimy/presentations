## はじめての Rust と

## Brainf\*\*k の処理系

<p style="text-align: left;">
@mozamimy
</p>

---

## 🐰誰? @mozamimy

<img src="./images/octorabbit_no_background.png" style="width: 150px;">

```sql
select * from rabbits where id = 'mozamimy'\G
```

```
*************************** 1. row ***************************
               id: mozamimy
          twitter: @mozamimy
     mail_address: alice@mozami.me
         web_site: https://mozami.me/
       working_on: Cookpad Inc.
        job_title: Site Reliability Engineer
   responsibility: エンジニアリングでインフラの運用をいい感じにする
           ᕱ⑅ᕱ ♡: AWS / Linux / Ruby / Rust / Vim
1 row in set (0.00 sec)
```

---

## Rust で実装した

## Brainf\*\*k の処理系の話を

## します

---

## 💪 モチベーション

- 言語処理系に興味
  - うさぎ言語作りたい..
- まずは素振りをする
  - {字句,構文}解析・コンパイラ・VM(スタックマシン)
  - 一通り実装したい
- Rust かっこいいし使ってみたい
- 数日程度でサクッとできる題材..🤔

---

# Brainf\*\*k

---

## Brainf\*\*k

- 難解プログラミング言語の一種
- テープ状のメモリの上をポインタが移動
- `>`: ポインタのインクリメント
- `<`: ポインタのデクリメント
- `+`: 指す値をインクリメント
- `-`: 指す値をデクリメント
- `.`: 指す値を出力
- `,`: 1 byte 読み込む
- `[`: 指す値が 0 なら対応する `]` の直後に移動
- `]`: 指す値が 0 でないなら対応する `[` に移動

---

# Kaguya 2

https://github.com/mozamimy/kaguya2

---

# 📟実装

---

## モジュール分割

- main.rs
- parser.rs
- ast.rs
- compiler.rs
- virtual_machine.rs

---

## 📟 main.rs

```rust
extern crate kaguya2;

use std::env;
use std::fs::File;
use std::io::Read;

use kaguya2::parser;
use kaguya2::ast;
use kaguya2::compiler;
use kaguya2::virtual_machine;

fn main() {
    let filepath = env::args().nth(1).unwrap();
    let mut file = File::open(&filepath).expect("Couldn't open file");

    let mut script = String::new();
    file.read_to_string(&mut script).expect("Couldn't read file");
    let script = script;

    let parser = parser::Parser::new(script);

    let arena = &mut ast::NodeArena { arena: Vec::new() };
    let root_id = arena.alloc(ast::NodeType::Root, None);

    parser.parse(root_id, arena);

    let compiler = compiler::Compiler::new(root_id, arena);
    let iseq = compiler.compile();

    let virtual_machine = &mut virtual_machine::VirtualMachine::new(iseq);
    virtual_machine.run();
}
```

---

## 📃 parser.rs・ast.rs

- `[` と `]` の対応がとれればよい
- `>>[--]+` の例

```
・[Root]
┣・[Forward (>)]
┣・[Forward (>)]
┣・[While]
┃┣・[Decrement (-)]
┃┗・[Decrement (-)]
┗・[Increment (+)]
```

---

## 🌲 ast.rs

- Arena pattern で木構造 (AST) を実装
- 走査は Visitor pattern で

```rust
use compiler;
use virtual_machine;

#[derive(Debug)]
pub enum NodeType {
    Root,
    Forward,
    Backward,
    Increment,
    Decrement,
    Output,
    Input,
    While,
}

#[derive(Debug)]
pub struct Node {
    pub node_id: NodeId,
    pub parent: Option<NodeId>,
    pub children: Vec<NodeId>,
    pub ntype: NodeType,
}

#[derive(Debug)]
pub struct NodeArena {
    pub arena: Vec<Node>,
}

pub type NodeId = usize;

impl NodeArena {
    pub fn alloc(&mut self, ntype: NodeType, parent: Option<NodeId>) -> NodeId {
        let id = self.arena.len();
        let node = Node { node_id: id, parent: parent, children: Vec::new(), ntype: ntype };
        self.arena.push(node);
        id
    }

    pub fn get(&self, id: NodeId) -> &Node {
        &self.arena[id]
    }

    pub fn get_mut(&mut self, id: NodeId) -> &mut Node {
        &mut self.arena[id]
    }

    pub fn append_child(&mut self, parent_id: NodeId, child_id: NodeId) {
        &self.get_mut(parent_id).children.push(child_id);
    }
}

impl Node {
    pub fn accept(&self, compiler: &compiler::Compiler) -> Vec<virtual_machine::Instruction> {
        compiler.visit(self.node_id)
    }
}
```

---

## 📃 parser.rs

- 1 文字ずつ読んでパタンマッチ

```rust
use ast;

#[derive(Debug)]
pub struct Parser {
    pub input: String,
}

impl Parser {
    pub fn new(input: String) -> Parser {
        Parser { input: input }
    }

    pub fn parse(&self, root_id: ast::NodeId, arena: &mut ast::NodeArena) {
        let mut current_node_id = Some(root_id);
        let mut context_level = 0;

        for chr in self.input.chars() {
            match chr {
                '>' => {
                    let new_node_id = arena.alloc(ast::NodeType::Forward, current_node_id);
                    arena.append_child(current_node_id.unwrap(), new_node_id);
                },
                '<' => {
                    let new_node_id = arena.alloc(ast::NodeType::Backward, current_node_id);
                    arena.append_child(current_node_id.unwrap(), new_node_id);
                },
                '+' => {
                    let new_node_id = arena.alloc(ast::NodeType::Increment, current_node_id);
                    arena.append_child(current_node_id.unwrap(), new_node_id);
                },
                '-' => {
                    let new_node_id = arena.alloc(ast::NodeType::Decrement, current_node_id);
                    arena.append_child(current_node_id.unwrap(), new_node_id);
                },
                '.' => {
                    let new_node_id = arena.alloc(ast::NodeType::Output, current_node_id);
                    arena.append_child(current_node_id.unwrap(), new_node_id);
                },
                ',' => {
                    let new_node_id = arena.alloc(ast::NodeType::Input, current_node_id);
                    arena.append_child(current_node_id.unwrap(), new_node_id);
                },
                '[' => {
                    let new_node_id = arena.alloc(ast::NodeType::While, current_node_id);
                    arena.append_child(current_node_id.unwrap(), new_node_id);
                    current_node_id = Some(new_node_id);
                    context_level += 1;
                },
                ']' => {
                    current_node_id = arena.get(current_node_id.unwrap()).parent;
                    match current_node_id {
                        None => panic!("Invalid brace correspondence."),
                        Some(_) => { /* noop */ },
                    }
                    context_level -= 1;
                },
                ' ' | '\n' | '\r' => {
                    // noop, read next character
                },
                _ => panic!("Invalid character: {}", chr),
            }
        }

        if context_level != 0 {
            panic!("Invalid brace correspondence.");
        }
    }
}
```

---

## 📟 Virtual Machine

- 実行対象の iseq (instcution sequence)
- プログラムカウンタ
- データ用のスタック 2 本

---

## 2 本のスタック

- Why?
  - ポインタのデクリメント時にデータを保持
  - ポインタの前後でぶったぎり

BF 上のメモリイメージ

```
          ↓
[0][1][1][2][1][3]
```

Kaguya VM 上のメモリイメージ

```
                       ↓
left_stack:  [0][1][1][2]
                 ↓
right_stack: [3][1]
```

---

## 命令の種類 (iseq)

- Forward (>)
- Backward (<)
- Increment (+)
- Decrement (-)
- Output (.)
- Input (,)
- BranchIfZero
- BranchUnlessZero
- Leave

---

## compiler.rs

- AST をコンパイルして iseq をつくる
- 例: `>[+-]-` のコンパイル結果

```
0: Forward, NULL
1: BranchIfZero, 4
2: Increment, NULL
3: Decrement, NULL
4: BranchUnlessZero, -2
5: Decrement, NULL
6: Leave, NULL

※ 番地: 命令, 引数1
```

---

## compiler.rs

```rust

use ast;
use virtual_machine;

#[derive(Debug)]
pub struct Compiler<'a> {
    ast_root_id: ast::NodeId,
    ast_arena: &'a mut ast::NodeArena,
}

impl<'a> Compiler<'a> {
    pub fn new(ast_root_id: ast::NodeId, ast_arena: &mut ast::NodeArena) -> Compiler {
        Compiler { ast_root_id: ast_root_id, ast_arena: ast_arena }
    }

    pub fn compile(&self) -> Vec<(virtual_machine::Instruction)> {
        let root = self.ast_arena.get(self.ast_root_id);
        let mut iseq = root.accept(self);
        iseq.push(virtual_machine::Instruction {
            instruction_type: virtual_machine::InstructionType::Leave,
            operand: None,
        });
        iseq
    }

    pub fn visit(&self, node_id: usize) -> Vec<virtual_machine::Instruction> {
        let mut iseq = Vec::new();
        let node = self.ast_arena.get(node_id);

        match node.ntype {
            ast::NodeType::Forward => {
                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::Forward,
                    operand: None,
                })
            },
            ast::NodeType::Backward => {
                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::Backward,
                    operand: None,
                })
            },
            ast::NodeType::Increment => {
                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::Increment,
                    operand: None,
                })
            },
            ast::NodeType::Decrement => {
                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::Decrement,
                    operand: None,
                })
            },
            ast::NodeType::Output => {
                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::Output,
                    operand: None,
                })
            },
            ast::NodeType::Input => {
                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::Input,
                    operand: None,
                })
            },
            ast::NodeType::While => {
                let mut sub_iseq = Vec::new();
                let children = node.children.clone();

                for i in children {
                    let child = self.ast_arena.get(i);
                    sub_iseq.append(&mut child.accept(self));
                }

                let sub_iseq_length = sub_iseq.len() as i32;

                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::BranchIfZero,
                    operand: Some(sub_iseq_length + 2),
                });
                iseq.append(&mut sub_iseq);
                iseq.push(virtual_machine::Instruction {
                    instruction_type: virtual_machine::InstructionType::BranchUnlessZero,
                    operand: Some(-sub_iseq_length),
                });
            },
            ast::NodeType::Root => {
                let children = node.children.clone();

                for i in children {
                    let child = self.ast_arena.get(i);
                    iseq.append(&mut child.accept(self));
                }
            },
        }

        iseq
    }
}
```

---

## virtual_machine.rs

- 粛々と実行するだけ

```rust
use std::process;
use libc::getchar;

#[derive(Debug, Clone)]
pub enum InstructionType {
    Forward,
    Backward,
    Increment,
    Decrement,
    Output,
    Input,
    BranchIfZero,
    BranchUnlessZero,
    Leave,
}

#[derive(Debug, Clone)]
pub struct Instruction {
    pub instruction_type: InstructionType,
    pub operand: Option<i32>,
}

#[derive(Debug)]
pub struct VirtualMachine {
    iseq: Vec<Instruction>,
    pc: u32,
    left_stack: Vec<u8>,
    right_stack: Vec<u8>,
}

impl VirtualMachine {
    pub fn new(iseq: Vec<Instruction>) -> VirtualMachine {
        VirtualMachine {
            iseq: iseq,
            pc: 0,
            left_stack: vec![0],
            right_stack: vec![],
        }
    }

    pub fn run(&mut self) {
        loop {
            let instruction = self.fetch(self.pc);
            self.execute(instruction);
        }
    }

    fn fetch(&self, pc: u32) -> Instruction {
        self.iseq[pc as usize].clone()
    }

    fn execute(&mut self, instruction: Instruction) {
        match instruction.instruction_type {
            InstructionType::Forward => {
                if self.right_stack.len() < 1 {
                    self.left_stack.push(0);
                } else {
                    self.left_stack.push(self.right_stack.pop().unwrap());
                }
                self.pc += 1;
            },
            InstructionType::Backward => {
                self.right_stack.push(self.left_stack.pop().unwrap());
                self.pc += 1;
            },
            InstructionType::Increment => {
                let new_value = self.left_stack.pop().unwrap() + 1;
                self.left_stack.push(new_value);
                self.pc += 1;
            },
            InstructionType::Decrement => {
                let new_value = self.left_stack.pop().unwrap() - 1;
                self.left_stack.push(new_value);
                self.pc += 1;
            },
            InstructionType::Output => {
                let value = self.left_stack.pop();
                print!("{}", value.unwrap() as char);
                self.left_stack.push(value.unwrap());
                self.pc += 1;
            },
            InstructionType::Input => {
                self.left_stack.pop();
                let value: u8;
                unsafe {
                    value = getchar() as u8;
                }
                self.left_stack.push(value);
                self.pc += 1;
            },
            InstructionType::BranchIfZero => {
                let value = self.left_stack.pop().unwrap();
                self.left_stack.push(value);

                if value == 0 {
                    self.pc = (self.pc as i32 + instruction.operand.unwrap()) as u32;
                } else {
                    self.pc += 1;
                }
            },
            InstructionType::BranchUnlessZero => {
                let value = self.left_stack.pop().unwrap();
                self.left_stack.push(value);

                if value != 0 {
                    self.pc = (self.pc as i32 + instruction.operand.unwrap()) as u32;
                } else {
                    self.pc += 1;
                }
            },
            InstructionType::Leave => {
                process::exit(0);
            },
        }
    }
}
```

---

## まとめ

- Rust 初心者でもできた 🙌
- 勘所はつかめた.. と思う
- 活用して仲良くしていきたい
  - ミドルウェアを作る?
  - うさぎ言語を作る?

---

## ボツスライド

---

## 💪 学び方

- [The Rust Programming Language](https://www.rust-lang.org/en-US/)
  - みんなだいすき至れり尽くせりドキュメント
  - 一通り写経
  - 所有権・ライフタイムは知識としてある程度知ってた
- Ruby で実装した Kaguya を移植
  - 差分で覚えたほうが速い

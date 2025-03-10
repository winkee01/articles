# My notes

## Utilities
Quit All                    | :qa
Terminal                    | :18sp|:term
Edit Config                 | :e ~/.config/nvim/init.lua

## t-mux @tmux
New Session                 | tmux new -s{{SESSION_NAME}}
List Sessions               | <prefix> s or tmux ls
Session and Window Preview  | <prefix> w
Rename Session              | <prefix> $
Detach from Session         | <prefix> d
Kill Session                | tmux kill-session -t{{SESSION_NAME}}
New Window                  | <prefix> c
Close Window                | <prefix> &
Rename Window               | <prefix> ,
Next Window                 | <prefix> n
Previous Window             | <prefix> p
Last Window                 | <prefix> l
Horizontal Pane             | <prefix> "
Vertical Pane               | <prefix> %
Close Pane                  | <prefix> x
Resize Pane                 | <prefix> <C-Up|Down|Left|Right>
Last Pane                   | <prefix> ;
Show Pane Numbers           | <prefix> q
Select Pane by Number       | <prefix> q 0..9
Move Pane Left|Right        | <prefix> {|}
Switch to Pane in direction | <prefix> Up|Down|Left|Rights

## json
Pretty Print                | :%!jq
Minify                      | :%!jq -r tostring

## lsp
Go to Definition            | gD
Hover                       | K
Implementations             | gi
References                  | gr
Document Symbol             | gd
Document Symbols            | gds
Workspace Symbols           | gws
Run                         | <leader> cl
Signature help              | <leader> sh
Rename                      | <leader> rn
Format                      | <leader> f
Code action                 | <leader> ca
Workspace Diagnostics       | <leader> aa
Workspace Errors            | <leader> ae
Workspace Warnings          | <leader> aw
Buffer Diagnostics          | <leader> d
Previous Diagnostic         | [c
Next Diagnostic             | ]c
Symbols Outline             | <leader> o

## Abolish/Coercion
Camel Case                  | crc
Pascal Case                 | crm
Snake Case                  | crs
Uppercase                   | cru
Kebab Case                  | crk
Dot case                    | cr.
Space Case                  | cr<space>
Title Case                  | crt

## Fold
Fold Close                  | zc
Preview Fold                | h
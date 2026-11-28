# Across Words — notas para trabalhar neste repositório

Jogo de palavras cruzadas em Godot 4.7 / GDScript. Quase toda a lógica
fica em um único arquivo: `main.gd` (~3400+ linhas). Não há testes
automatizados; a validação é feita rodando o próprio jogo (headless e,
quando o bug é visual, com renderização real via Xvfb).

## Antes de rodar o jogo (sempre, num clone novo)

Os ícones SVG (`assets/icons/*.svg`) e o favicon precisam ser importados
pelo editor antes de qualquer execução, senão aparecem erros
`No loader found for resource: res://assets/icons/*.svg`:

```
godot4 --headless --path . --import
```

(Nesta máquina o binário é `/home/claude/Godot_v4.7-stable_linux.x86_64`.)
Os arquivos `*.svg.import` gerados e o cache em `.godot/` **não** devem
ser commitados — já estão (quase todos) no `.gitignore`; se aparecerem
como `modified`/`??` no `git status`, descarte antes de commitar
(`git checkout -- .godot/... && rm -rf assets/icons/*.svg.import web/favicon.svg.import`).

## Validar uma mudança

1. **Checagem rápida (sempre)** — garante que não há erro de parse/script:
   ```
   godot4 --headless --path . --quit-after 180
   ```
2. **Checagem visual (bugs de renderização, seleção, animação)** — sobe um
   display virtual com renderização por software e tira screenshots reais:
   ```
   Xvfb :99 -screen 0 1280x960x24 &
   export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
   godot4 --path . --rendering-driver opengl3 --rendering-method gl_compatibility
   ```
   Para automatizar um teste (ex.: selecionar âncora, digitar, confirmar),
   adicione TEMPORARIAMENTE em `_ready()` algo como
   `if OS.get_environment("AW_DEBUG_X") == "1": _debug_x()`, com a função
   de teste chamando `await RenderingServer.frame_post_draw` +
   `get_viewport().get_texture().get_image().save_png(...)` para capturar
   cada passo. **Sempre remova esse código antes de commitar** — confira
   com `grep -n DEBUG main.gd` e comparando o md5 do arquivo com o do HEAD.

## Publicar (branch → PR → merge → Pages)

O usuário pediu que, a partir de agora, toda mudança seja automaticamente
commitada, mergeada e publicada — sem precisar pedir cada etapa:

```
git checkout -b fix/descricao-curta
git commit -m "..."            # ver trailer obrigatório abaixo
git push -u origin fix/descricao-curta

# gh pr create falha (GraphQL bloqueado nesta sessão) — usar REST:
gh api repos/valusuelo/acrosswords/pulls -f title='...' -f head='fix/...' -f base='main' -f body='...'

# esperar mergeable_state=clean, depois:
gh api -X PUT repos/valusuelo/acrosswords/pulls/<N>/merge -f merge_method=squash -f commit_title='...'

git checkout main && git pull origin main
gh workflow run "Publicar jogo no GitHub Pages" -R valusuelo/acrosswords --ref main
gh run watch <run-id> -R valusuelo/acrosswords --exit-status
```

Jogo publicado em: https://valusuelo.github.io/acrosswords/
Outros workflows disponíveis (mesma lógica, `gh workflow run "<nome>" -R valusuelo/acrosswords --ref main`):
"Compilar jogo para computador", "Gerar APK Android (teste)".

Toda mensagem de commit e PR termina com (ver instruções da sessão para o
link de sessão atual — não reutilizar um antigo):
```
Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: <link da sessão atual>
```

## Mapa rápido da mecânica de seleção (main.gd)

- `words: Array[Dictionary]` — cada palavra tem `answer`, `dir` (Vector2i),
  `start` (Vector2i), `state` ("active"/"solved"/...). `_cells(word)` dá as
  células ocupadas.
- `preview_parent`/`preview_anchor`/`preview_dir` — âncora e direção sendo
  usadas para ramificar uma nova palavra a partir de uma já resolvida.
  `last_solved_word` é a palavra atualmente destacada em dourado.
- `_select_anchor(index, anchor)` → `_load_preview_choices()` →
  `_possible_children()` (busca no `words_by_initial`, filtra por
  `_occupied_letters()` e `_word_difficulty_allowed()`) → `_show_preview_choice()`.
- `_click_tile(p, entry_direction)`, `_solved_words_at_cell(cell)` e
  `_branch_side_directions(anchor_cell)` tratam o caso de "canto" (célula
  pertence a duas palavras cruzadas) — usam a direção de chegada do
  arrasto/seta para desempatar qual palavra fica ativa.
- `_refresh_board()` recalcula `tile_state` por célula e aplica cores;
  `_queue_stage_open_tiles(cells, from_abyss)` anima blocos novos emergindo.
- `data/words.json` é o banco de palavras. Entradas cuja dica termina em
  `"(pl.)"` são variantes no plural e são filtradas no load
  (`_load_word_bank`) — não devem voltar a ser usadas como candidatas.

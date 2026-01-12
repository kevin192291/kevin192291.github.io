<script>
  import { onMount } from 'svelte';

  const BLOCK_W = 230;
  const BLOCK_H = 96;

  const typeMeta = {
    value: { title: 'Value', color: 'var(--accent)', editable: true, linkOut: false },
    pointer: { title: 'Pointer', color: 'var(--teal)', editable: false, linkOut: true },
    pointer2: { title: 'Pointer*2', color: 'var(--cyan)', editable: false, linkOut: true },
    funcptr: { title: 'Func Ptr', color: 'var(--olive)', editable: false, linkOut: true },
    function: { title: 'Function', color: 'var(--rose)', editable: false, linkOut: false },
    array: { title: 'Array', color: 'var(--accent-strong)', editable: true, linkOut: false },
    struct: { title: 'Struct', color: 'var(--teal)', editable: true, linkOut: false }
  };

  let canvasEl = $state(null);
  let canvasRect = $state({ left: 0, top: 0, width: 0, height: 0 });

  let blocks = $state([
    { id: 'val_count', type: 'value', label: 'count', name: 'count', addr: 4096, value: 42, x: 32, y: 36 },
    { id: 'val_flag', type: 'value', label: 'flag', name: 'flag', addr: 4100, value: 1, x: 32, y: 160 },
    { id: 'ptr_p', type: 'pointer', label: 'p', name: 'p', addr: 4104, value: 0, x: 320, y: 36 },
    { id: 'ptr_pp', type: 'pointer2', label: 'pp', name: 'pp', addr: 4108, value: 0, x: 320, y: 160 },
    { id: 'fn_square', type: 'function', label: 'square', name: 'square', addr: 8192, value: 0, x: 32, y: 284 },
    { id: 'fn_ptr', type: 'funcptr', label: 'fp', name: 'fp', addr: 4112, value: 0, x: 320, y: 284 },
    { id: 'arr_nums', type: 'array', label: 'nums[3]', name: 'nums', addr: 4120, value: 5, x: 32, y: 408 },
    { id: 'struct_pair', type: 'struct', label: 'pair', name: 'pair', addr: 4132, value: 2, x: 320, y: 408 }
  ]);

  let links = $state([
    { id: 'link-p', from: 'ptr_p', to: 'val_count' },
    { id: 'link-pp', from: 'ptr_pp', to: 'ptr_p' },
    { id: 'link-fp', from: 'fn_ptr', to: 'fn_square' }
  ]);

  let dragging = $state(null);
  let linking = $state(null);
  let hoveredTarget = $state(null);

  const clamp = (value, min, max) => Math.min(max, Math.max(min, value));
  const getBlock = (id) => blocks.find((block) => block.id === id);
  const getLink = (fromId) => links.find((link) => link.from === fromId);

  const formatHex = (value) => `0x${Math.max(0, value).toString(16).toUpperCase()}`;
  const formatBin = (value) => {
    const raw = Math.max(0, value >>> 0).toString(2).padStart(16, '0');
    return `0b${raw}`;
  };

  const resolveValue = (block) => {
    if (typeMeta[block.type].editable) {
      return Number.isFinite(block.value) ? block.value : 0;
    }

    if (block.type === 'function') {
      return block.addr;
    }

    const link = getLink(block.id);
    if (!link) {
      return 0;
    }

    const target = getBlock(link.to);
    return target ? target.addr : 0;
  };

  const handleValueInput = (block, event) => {
    const next = Number.parseInt(event.currentTarget.value, 10);
    block.value = Number.isNaN(next) ? 0 : next;
  };

  const updateCanvasRect = () => {
    if (!canvasEl) {
      return;
    }
    const rect = canvasEl.getBoundingClientRect();
    canvasRect = { left: rect.left, top: rect.top, width: rect.width, height: rect.height };
  };

  const startDrag = (event, block) => {
    if (event.button !== 0) {
      return;
    }
    if (event.target.closest('.connector') || event.target.tagName === 'INPUT') {
      return;
    }
    updateCanvasRect();
    dragging = {
      id: block.id,
      offsetX: event.clientX - canvasRect.left - block.x,
      offsetY: event.clientY - canvasRect.top - block.y
    };
  };

  const startLink = (event, block) => {
    event.stopPropagation();
    updateCanvasRect();
    linking = {
      fromId: block.id,
      x: event.clientX - canvasRect.left,
      y: event.clientY - canvasRect.top
    };
  };

  const connectorPosition = (block, side) => {
    return {
      x: block.x + (side === 'out' ? BLOCK_W : 0),
      y: block.y + BLOCK_H / 2
    };
  };

  const findTargetAt = (x, y, fromId) => {
    const radius = 16;
    for (const block of blocks) {
      if (block.id === fromId) {
        continue;
      }
      const pos = connectorPosition(block, 'in');
      const dx = x - pos.x;
      const dy = y - pos.y;
      if (Math.hypot(dx, dy) <= radius) {
        return block.id;
      }
    }
    return null;
  };

  const finishLink = () => {
    if (!linking) {
      return;
    }
    if (hoveredTarget) {
      links = links.filter((link) => link.from !== linking.fromId);
      links = [
        ...links,
        { id: `link-${linking.fromId}-${hoveredTarget}`, from: linking.fromId, to: hoveredTarget }
      ];
    }
    linking = null;
    hoveredTarget = null;
  };

  const updatePointer = (event) => {
    if (dragging) {
      const block = getBlock(dragging.id);
      if (block) {
        const maxX = Math.max(0, canvasRect.width - BLOCK_W - 8);
        const maxY = Math.max(0, canvasRect.height - BLOCK_H - 8);
        block.x = clamp(event.clientX - canvasRect.left - dragging.offsetX, 8, maxX);
        block.y = clamp(event.clientY - canvasRect.top - dragging.offsetY, 8, maxY);
      }
    }

    if (linking) {
      const nextX = event.clientX - canvasRect.left;
      const nextY = event.clientY - canvasRect.top;
      linking.x = nextX;
      linking.y = nextY;
      hoveredTarget = findTargetAt(nextX, nextY, linking.fromId);
    }
  };

  const stopPointer = () => {
    dragging = null;
    finishLink();
  };

  onMount(() => {
    updateCanvasRect();
    const handleMove = (event) => updatePointer(event);
    const handleUp = () => stopPointer();
    const handleResize = () => updateCanvasRect();

    window.addEventListener('pointermove', handleMove);
    window.addEventListener('pointerup', handleUp);
    window.addEventListener('resize', handleResize);
    window.addEventListener('scroll', handleResize, true);

    const resizeObserver = new ResizeObserver(() => updateCanvasRect());
    if (canvasEl) {
      resizeObserver.observe(canvasEl);
    }

    return () => {
      window.removeEventListener('pointermove', handleMove);
      window.removeEventListener('pointerup', handleUp);
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('scroll', handleResize, true);
      resizeObserver.disconnect();
    };
  });

  const linkPaths = $derived.by(() => {
    return links
      .map((link) => {
        const from = getBlock(link.from);
        const to = getBlock(link.to);
        if (!from || !to) {
          return null;
        }
        const start = connectorPosition(from, 'out');
        const end = connectorPosition(to, 'in');
        const bend = Math.max(80, Math.abs(end.x - start.x) * 0.4);
        const c1x = start.x + bend;
        const c2x = end.x - bend;
        const d = `M ${start.x} ${start.y} C ${c1x} ${start.y}, ${c2x} ${end.y}, ${end.x} ${end.y}`;
        return { id: link.id, d, color: typeMeta[from.type].color };
      })
      .filter(Boolean);
  });

  const draftPath = $derived.by(() => {
    if (!linking) {
      return null;
    }
    const from = getBlock(linking.fromId);
    if (!from) {
      return null;
    }
    const start = connectorPosition(from, 'out');
    const end = { x: linking.x, y: linking.y };
    const bend = Math.max(80, Math.abs(end.x - start.x) * 0.4);
    const c1x = start.x + bend;
    const c2x = end.x - bend;
    return `M ${start.x} ${start.y} C ${c1x} ${start.y}, ${c2x} ${end.y}, ${end.x} ${end.y}`;
  });

  const memoryCells = $derived.by(() => {
    return [...blocks]
      .map((block) => ({
        ...block,
        resolvedValue: resolveValue(block)
      }))
      .sort((a, b) => b.addr - a.addr);
  });

  const codeLines = $derived.by(() => {
    const lines = [];

    const tok = (cls, text) => ({ cls, text });
    const plain = (text) => ({ cls: '', text });

    const valueBlocks = blocks.filter((block) => block.type === 'value');
    const arrayBlocks = blocks.filter((block) => block.type === 'array');
    const structBlocks = blocks.filter((block) => block.type === 'struct');
    const functionBlocks = blocks.filter((block) => block.type === 'function');
    const pointerBlocks = blocks.filter((block) => block.type === 'pointer');
    const pointer2Blocks = blocks.filter((block) => block.type === 'pointer2');
    const funcPtrBlocks = blocks.filter((block) => block.type === 'funcptr');

    for (const block of valueBlocks) {
      lines.push([
        tok('tok-type', 'int'),
        plain(' '),
        tok('tok-name', block.name),
        plain(' = '),
        tok('tok-num', String(block.value)),
        plain(';')
      ]);
    }

    for (const block of arrayBlocks) {
      const base = Number.isFinite(block.value) ? block.value : 0;
      lines.push([
        tok('tok-type', 'int'),
        plain(' '),
        tok('tok-name', `${block.name}[3]`),
        plain(' = { '),
        tok('tok-num', String(base)),
        plain(', '),
        tok('tok-num', String(base + 1)),
        plain(', '),
        tok('tok-num', String(base + 2)),
        plain(' };')
      ]);
    }

    if (structBlocks.length > 0) {
      lines.push([
        tok('tok-kw', 'struct'),
        plain(' '),
        tok('tok-name', 'Pair'),
        plain(' { '),
        tok('tok-type', 'int'),
        plain(' left; '),
        tok('tok-type', 'int'),
        plain(' right; };')
      ]);

      for (const block of structBlocks) {
        const left = Number.isFinite(block.value) ? block.value : 0;
        const right = left + 1;
        lines.push([
          tok('tok-kw', 'struct'),
          plain(' '),
          tok('tok-name', 'Pair'),
          plain(' '),
          tok('tok-name', block.name),
          plain(' = { '),
          tok('tok-num', String(left)),
          plain(', '),
          tok('tok-num', String(right)),
          plain(' };')
        ]);
      }
    }

    for (const block of functionBlocks) {
      lines.push([
        tok('tok-type', 'int'),
        plain(' '),
        tok('tok-fn', block.name),
        plain('(int x) {')
      ]);
      lines.push([
        plain('  '),
        tok('tok-kw', 'return'),
        plain(' '),
        tok('tok-name', 'x'),
        plain(' '),
        tok('tok-op', '*'),
        plain(' '),
        tok('tok-name', 'x'),
        plain(';')
      ]);
      lines.push([plain('}')]);
    }

    const pointerLine = (block, stars) => {
      const link = getLink(block.id);
      const target = link ? getBlock(link.to) : null;
      let targetExpr = 'NULL';
      if (target) {
        if (target.type === 'array') {
          targetExpr = target.name;
        } else if (target.type === 'function') {
          targetExpr = `&${target.name}`;
        } else {
          targetExpr = `&${target.name}`;
        }
      }

      lines.push([
        tok('tok-type', 'int'),
        plain(' '),
        tok('tok-op', '*'.repeat(stars)),
        tok('tok-name', block.name),
        plain(' = '),
        targetExpr === 'NULL' ? tok('tok-kw', 'NULL') : tok('tok-name', targetExpr),
        plain(';')
      ]);
    };

    for (const block of pointerBlocks) {
      pointerLine(block, 1);
    }

    for (const block of pointer2Blocks) {
      pointerLine(block, 2);
    }

    for (const block of funcPtrBlocks) {
      const link = getLink(block.id);
      const target = link ? getBlock(link.to) : null;
      const targetName = target ? target.name : 'NULL';
      lines.push([
        tok('tok-type', 'int'),
        plain(' '),
        tok('tok-op', '(*'),
        tok('tok-name', block.name),
        tok('tok-op', ')'),
        plain('(int)'),
        plain(' = '),
        targetName === 'NULL' ? tok('tok-kw', 'NULL') : tok('tok-name', targetName),
        plain(';')
      ]);
    }

    return lines;
  });
</script>

<main>
  <div class="page">
    <div>
      <section class="section hero">
        <div class="tagline">Pointers Lab</div>
        <h1>Drag, link, and decode how C and C++ pointers actually behave.</h1>
        <p>
          Each block is a variable with a memory address and a value. Drag blocks around the
          workspace, then drag a link from pointer blocks to whatever they should point at.
        </p>
        <div class="callout">
          <div class="pill">Drag blocks to move them</div>
          <div class="pill">Drag connectors to link pointers</div>
          <div class="pill">Edit values to watch memory update</div>
        </div>
      </section>

      <section class="section playground">
        <h2>Pointer playground</h2>
        <p>
          Pointer blocks output the address of their target. Pointer*2 blocks output the address of a
          pointer. Function pointers can target the function block below. Array and struct blocks are
          extra data shapes for practice.
        </p>

        <div class="canvas" bind:this={canvasEl}>
          <svg class="link-layer" viewBox={`0 0 ${canvasRect.width} ${canvasRect.height}`} preserveAspectRatio="none">
            <defs>
              <marker
                id="arrow"
                markerWidth="10"
                markerHeight="10"
                refX="8"
                refY="3"
                orient="auto"
                markerUnits="strokeWidth"
              >
                <path d="M0,0 L0,6 L9,3 z" fill="#1d1b19" />
              </marker>
            </defs>
            {#each linkPaths as link (link.id)}
              <path
                d={link.d}
                stroke={link.color}
                stroke-width="2.2"
                fill="none"
                marker-end="url(#arrow)"
              />
            {/each}
            {#if draftPath}
              <path d={draftPath} stroke="#1d1b19" stroke-width="2" fill="none" stroke-dasharray="6 6" />
            {/if}
          </svg>

          {#each blocks as block (block.id)}
            <div
              class={`block ${hoveredTarget === block.id ? 'is-target' : ''}`}
              style={`transform: translate(${block.x}px, ${block.y}px); --block-color: ${typeMeta[block.type].color};`}
              onpointerdown={(event) => startDrag(event, block)}
            >
              <div class="block-header">
                <span class="block-type">{typeMeta[block.type].title}</span>
                <span class="block-label">{block.label}</span>
              </div>
              <div class="block-body">
                <div>
                  <div class="addr">{formatHex(block.addr)}</div>
                  <div class="value-meta">{formatBin(block.addr)}</div>
                </div>
                <div class="value">
                  {#if typeMeta[block.type].editable}
                    <input type="number" value={block.value} oninput={(event) => handleValueInput(block, event)} />
                  {:else}
                    <input type="text" readonly value={resolveValue(block)} />
                  {/if}
                  <div class="value-meta">
                    <span>{formatHex(resolveValue(block))}</span>
                    <span>{formatBin(resolveValue(block))}</span>
                  </div>
                </div>
              </div>
              <div class="connector in"></div>
              {#if typeMeta[block.type].linkOut}
                <div class="connector out" onpointerdown={(event) => startLink(event, block)}></div>
              {/if}
            </div>
          {/each}
        </div>

        <div class="legend">
          {#each Object.entries(typeMeta) as [type, meta] (type)}
            <span style={`--block-color: ${meta.color};`}>
              <i></i>
              {meta.title}
            </span>
          {/each}
        </div>

        <div class="code-window" aria-label="C code preview">
          {#each codeLines as line, lineIndex (lineIndex)}
            <div class="code-line">
              {#each line as token, tokenIndex (tokenIndex)}
                <span class={token.cls}>{token.text}</span>
              {/each}
            </div>
          {/each}
        </div>
      </section>

      <section class="section">
        <h2>Pointer instincts to build</h2>
        <div class="note-list">
          <div class="note">NULL means a pointer holds 0 and points nowhere.</div>
          <div class="note">Pointer*2 blocks let you chain indirection and visualize levels.</div>
          <div class="note">Arrays decay to pointers, so linking to an array uses its base address.</div>
          <div class="note">Function pointers store a code address, not a value on the stack.</div>
        </div>
      </section>
    </div>

    <aside class="memory-column">
      <div class="memory-panel">
        <div class="memory-title">Memory stack view</div>
        <div class="memory-stack">
          {#each memoryCells as cell, index (cell.id)}
            <div class="memory-cell" style={`--block-color: ${typeMeta[cell.type].color}; animation-delay: ${index * 40}ms;`}>
              <div class="cell-header">
                <span class="memory-chip">{typeMeta[cell.type].title}</span>
                <span>{formatHex(cell.addr)}</span>
              </div>
              <div class="cell-values">
                <span>dec: {cell.resolvedValue}</span>
                <span>hex: {formatHex(cell.resolvedValue)}</span>
                <span>bin: {formatBin(cell.resolvedValue)}</span>
              </div>
            </div>
          {/each}
        </div>
      </div>
    </aside>
  </div>
</main>

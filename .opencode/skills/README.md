# OpenCode Skills

Skills are maintained centrally in [7K-Hiroba/Hiroba](https://github.com/7K-Hiroba/Hiroba) under `.opencode/skills/`.

This directory is a placeholder. To install skills as symlinks in your local clone, run:

```bash
git clone https://github.com/7K-Hiroba/Hiroba /tmp/hiroba
for skill in /tmp/hiroba/.opencode/skills/*/; do
  ln -sf "$skill" .opencode/skills/
done
```

Do not commit the symlinks — they are local-only and depend on the Hiroba clone path.

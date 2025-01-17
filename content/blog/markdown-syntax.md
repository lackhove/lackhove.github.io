+++
title = "Selfhosting with OpenSUSE MicroOs and Podman rootles containers"
date = "2024-01-16"
description = "My selfhosting eneadvours."
taxonomies.tags = [
    "selfhosting",
    "linux",
]
+++

## The Beginnings

I have been selfhosting since i received my Raspberry Pi 1 from the very first batch, mostly SSH and NFS to access a 3.5" HDD attached via USB.

```python

class ShellFileReporter(FileReporter):
    def __init__(self, filename: str) -> None:
        super().__init__(filename)

        self.path = Path(filename)
        self._content: str | None = None
        self._executable_lines: set[int] = set()
        self._parser = get_parser("bash")

    def source(self) -> str:
        if self._content is None:
            if not self.path.is_file():
                return ""
            try:
                self._content = self.path.read_text()
            except UnicodeDecodeError:
                return ""

        return self._content

    def _parse_ast(self, node: Node) -> None:
        if node.is_named and node.type in EXECUTABLE_NODE_TYPES:
            self._executable_lines.add(node.start_point[0] + 1)

        for child in node.children:
            self._parse_ast(child)

    def lines(self) -> set[TLineNo]:
        tree = self._parser.parse(self.source().encode("utf-8"))
        self._parse_ast(tree.root_node)

        return self._executable_lines

```
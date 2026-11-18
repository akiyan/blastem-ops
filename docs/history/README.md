# 設計記録

本体へ統合する前、MCP サーバーを別リポジトリとして開発していた時期の計画・判断の記録。当時の構成（BlastEM を Git submodule で固定し、MCP を別リポジトリに置く形）を前提に書かれているため、手順は現在の構成と異なる。現在の構成は [README](../../README.md) と [導入](../setup.md) を参照する。

- [MCP 実装計画](mcp-plan.md)
- [固定した上流 fork の互換性の確認](compatibility.md)
- [ADR 0001: ulalume fork を使う](adr/0001-use-ulalume-blastem.md)
- [ADR 0002: MCP サーバーを Go で実装する](adr/0002-use-go-for-the-mcp-server.md)

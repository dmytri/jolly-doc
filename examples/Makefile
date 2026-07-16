# Generate diagram renderings for all formats
# Usage: make all     — render everything
#        make view    — render + view (requires chafa)
#        make clean   — remove generated files

OUTDIR = /tmp/jolly-renders
mkdir = mkdir -p $(OUTDIR)

all: dot-ascii dot-svg nomnoml-svg

# Graphviz DOT — ASCII output
dot-ascii: dot/jolly-context.dot
	dot -Tascii $< > $(OUTDIR)/jolly-context-ascii.txt
	@echo "=== DOT ASCII output ==="
	@cat $(OUTDIR)/jolly-context-ascii.txt

# Graphviz DOT — SVG output (colour)
dot-svg: dot/jolly-context.dot
	dot -Tsvg $< > $(OUTDIR)/jolly-context.svg
	@echo "DOT SVG: $(OUTDIR)/jolly-context.svg"

# Nomnoml — SVG output
nomnoml-svg: nomnoml/jolly-context.nomnoml
	cd nomnoml && npx nomnoml jolly-context.nomnoml $(OUTDIR)/jolly-nomnoml.svg
	@echo "Nomnoml SVG: $(OUTDIR)/jolly-nomnoml.svg"

# D2 — SVG output (requires d2)
d2-svg: d2/jolly-context.d2
	d2 --layout=elk $< $(OUTDIR)/jolly-d2.svg
	@echo "D2 SVG: $(OUTDIR)/jolly-d2.svg"

# View with chafa (requires `sudo apt install chafa`)
view: all
	chafa $(OUTDIR)/jolly-nomnoml.svg
	chafa $(OUTDIR)/jolly-context.svg

# Clean
clean:
	rm -rf $(OUTDIR)

.PHONY=clean all

%.html: %.org
	emacs $< --batch -l ~/.emacs.d/init.el -f org-html-export-to-html --kill

all: coverage.html

clean:
	rm -f *.html

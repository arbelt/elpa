# Makefile for GNU Emacs Lisp Package Archive.

EMACS=emacs

ARCHIVE_TMP=archive-tmp
SITE_DIR=site

.PHONY: archive-tmp changelogs process-archive archive-full org-fetch clean all do-it

all: all-in-place

## Set up the source files for direct usage, by pointing
## `package-directory-list' to the site/ directory.
site: packages
	mkdir -p $(SITE_DIR)
	$(EMACS) -batch -l $(CURDIR)/admin/archive-contents.el \
	  --eval "(batch-make-site-dir \"packages\" \"$(SITE_DIR)\")"

site/%: do-it
	$(EMACS) -batch -l $(CURDIR)/admin/archive-contents.el \
	  --eval "(progn (setq debug-on-error t) (batch-make-site-package \"$@\"))"

## Deploy the package archive to archive/, with packages in
## archive/packages/:
archive: archive-tmp
	$(MAKE) $(MFLAGS) process-archive

archive-tmp: packages changelogs
	-rm -r $(ARCHIVE_TMP)
	mkdir -p $(ARCHIVE_TMP)
	cp -a packages/. $(ARCHIVE_TMP)/packages

# Refresh the ChangeLog files.  This needs to be done in
# the source tree, because it needs the Bzr data!
changelogs:
	cd packages; \
	$(EMACS) -batch -l $(CURDIR)/admin/archive-contents.el \
			-f batch-prepare-packages

process-archive:
	# FIXME, we could probably speed this up significantly with
	# rules like "%.tar: ../%/ChangeLog" so we only rebuild the packages
	# that have indeed changed.
	cd $(ARCHIVE_TMP)/packages; $(EMACS) -batch -l $(CURDIR)/admin/archive-contents.el -f batch-make-archive
	@cd $(ARCHIVE_TMP)/packages; \
	for pt in *; do \
	    if [ -d $$pt ]; then \
		echo "Creating tarball $${pt}.tar" && \
		tar -cf $${pt}.tar $$pt --remove-files; \
	    fi; \
	done
	mkdir -p archive/packages
	mv archive/packages archive/packages-old
	mv $(ARCHIVE_TMP)/packages archive/packages
	chmod -R a+rX archive/packages
	rm -rf archive/packages-old
	rm -rf $(ARCHIVE_TMP)

## Deploy the package archive to archive/ including the Org daily:
archive-full: archive-tmp org-fetch
	$(MAKE) $(MFLAGS) process-archive
	#mkdir -p archive/admin
	#cp admin/* archive/admin/

org-fetch: archive-tmp
	cd $(ARCHIVE_TMP)/packages; \
	pkgname=`curl -s http://orgmode.org/elpa/|perl -ne 'push @f, $$1 if m/(org-\d{8})\.tar/; END { @f = sort @f; print "$$f[-1]\n"}'`; \
	wget -q http://orgmode.org/elpa/$${pkgname}.tar -O $${pkgname}.tar; \
	if [ -f $${pkgname}.tar ]; then \
		tar xf $${pkgname}.tar; \
		rm -f $${pkgname}.tar; \
		mv $${pkgname} org; \
	fi

clean:
	rm -rf archive $(ARCHIVE_TMP) $(SITE_DIR)

########## Rules for in-place installation ##########
pkgs := $(foreach pkg, $(wildcard packages/*), \
          $(if $(shell [ -d "$(pkg)" ] && echo true), $(pkg)))

define SET-diff
$(shell echo "$(1)" "$(2)" "$(2)" | tr ' ' '\n' | sort | uniq -u)
endef

define FILTER-nonsrc
$(filter-out %-autoloads.el %-pkg.el, $(1))
endef

define RULE-srcdeps
$(1): $$(call FILTER-nonsrc, $$(wildcard $$(dir $(1))/*.el))
endef

# Compute the set of autolods files and their dependencies.
autoloads := $(foreach pkg, $(pkgs), $(pkg)/$(notdir $(pkg))-autoloads.el)

$(foreach al, $(autoloads), $(eval $(call RULE-srcdeps, $(al))))
%-autoloads.el:
	@echo 'EMACS -f package-generate-autoloads $@'
	@cd $(dir $@); \
	$(EMACS) --batch \
	    -l $(CURDIR)/admin/archive-contents.el \
	    --eval "(archive--refresh-pkg-file)" \
	    --eval "(require 'package)" \
	    --eval "(package-generate-autoloads '$$(basename $$(pwd)) \
	                                        \"$$(pwd)\")"

# Put into elcs the set of elc files we need to keep up-to-date.
# I.e. one for each .el file except for the -pkg.el, the -autoloads.el, and
# the .el files that are marked "no-byte-compile".
els := $(call FILTER-nonsrc, $(wildcard packages/*/*.el))
naive_elcs := $(patsubst %.el, %.elc, $(els))
current_elcs := $(wildcard packages/*/*.elc)

extra_els := $(call SET-diff, $(els), $(patsubst %.elc, %.el, $(current_elcs)))
nbc_els := $(foreach el, $(extra_els), \
             $(if $(shell grep '^;.*no-byte-compile: t' "$(el)"), $(el)))
elcs := $(call SET-diff, $(naive_elcs), $(patsubst %.el, %.elc, $(nbc_els)))

# '(dolist (al (quote ($(patsubst %, "%", $(autoloads))))) (load (expand-file-name al) nil t))'
%.elc: %.el
	@echo 'EMACS -f batch-byte-compile $<'
	@$(EMACS) --batch \
	    --eval "(setq package-directory-list '(\"$(abspath packages)\"))" \
	    --eval '(package-initialize)' \
	    -L $(dir $@) -f batch-byte-compile $<

.PHONY: elcs
elcs: $(elcs)

# Remove .elc files that don't have a corresponding .el file any more.
extra_elcs := $(call SET-diff, $(current_elcs), $(naive_elcs))
.PHONY: $(extra_elcs)
$(extra_elcs):; rm $@

# # Put into single_pkgs the set of -pkg.el files we need to keep up-to-date.
# # I.e. all the -pkg.el files for the single-file packages.
# single_pkgs:=$(foreach pkg, $(pkgs), \
#                $(word $(words $(call FILTER-nonsrc, \
#                                      $(wildcard $(pkg)/*.el))), \
#                   $(pkg)/$(notdir $(pkg))-pkg.el))
# #$(foreach al, $(single_pkgs), $(eval $(call RULE-srcdeps, $(al))))
# %-pkg.el: %.el
# 	@echo 'EMACS -f package-generate-description-file $@'
# 	@$(EMACS) --batch \
# 	    --eval '(require (quote package))' \
# 	    --eval '(setq b (find-file-noselect "$<"))' \
# 	    --eval '(setq d (with-current-buffer b (package-buffer-info)))' \
# 	    --eval '(package-generate-description-file d "$(dir $@)")'

.PHONY: all-in-place
all-in-place: $(extra_elcs) $(autoloads) # $(single_pkgs)
	# Do them in a sub-make, so that autoloads are done first.
	$(MAKE) elcs
# From the Build Scripts Written By: Jim Gifford <lfs@jg555.com>
# Modified By: Joe Ciccone <jciccone@linuxfromscratch.org>
# Additional changes: George Boudreau <georgeb@linuxfromscratch.org>
#                     Pierre Labastie <pierre.labastie at neuf.fr>

ifdef V
  Q =
else
  Q = @
endif

# Makefile should reside in a directory where there are two subdirectories
# initially:
TOPDIR           = $(shell pwd)
# the stylesheets
XSLDIR           = $(TOPDIR)/xsl
# the menu program sources
MENU             = $(TOPDIR)/menu

# Those directories and files will be created and populated by make:
# directory of the book sources:
LFS_XML          = $(TOPDIR)/lfs-xml
BLFS_XML         = $(TOPDIR)/blfs-xml
# contains the REV used in the preceding call:
REVFILE          = $(TOPDIR)/revision
# the list of packages:
PACK_LIST        = $(TOPDIR)/packages.xml
# the generated menu input:
CONFIG_CONFIG_IN = $(TOPDIR)/Config.in
# menu output:
CONFIG_OUT       = $(TOPDIR)/configuration
# the linear book:
BOOK_XML         = $(TOPDIR)/book.xml

LFSTMP           = $(LFS_XML)/tmp
RENDERTMP        = $(BLFS_XML)/tmp
LFS_FULL         = $(LFSTMP)/lfs-full.xml
BLFS_FULL        = $(RENDERTMP)/blfs-full.xml

# The right-hand side is updated by jhalfs:
# Where the tracking file resides:
TRACKING_DIR = tracking-dir

# Will be created  by make, if not existent
TRACKFILE        = $(TRACKING_DIR)/instpkg.xml

# Initial content of the tracking file
define INITIAL_TRACK
<?xml version="1.0" encoding="ISO-8859-1"?>\n\
\n\
<!DOCTYPE sublist SYSTEM "$(TOPDIR)/packdesc.dtd">\n\
<sublist>\n\
  <name>Installed</name>\n\
</sublist>
endef

# Git locations and branches
LFS-GIT = https://git.linuxfromscratch.org/lfs.git
LFS-BRANCH = trunk
GIT = https://git.linuxfromscratch.org/blfs.git
BLFS-BRANCH = trunk

ALLXML := $(filter-out $(RENDERTMP)/%, \
              $(shell if [ -d $(BLFS_XML) ]; then \
                        find $(BLFS_XML) -name \*.xml; \
                      fi))
ALLXMLLFS:= $(filter-out $(LFSTMP)/%, \
              $(shell if [ -d $(LFS_XML) ]; then \
                        find $(LFS_XML) -name \*.xml; \
                      fi))
ALLXSL := $(filter-out $(RENDERTMP)/%, \
              $(shell if [ -d $(BLFS_XML) ]; then \
                        find $(BLFS_XML) -name \*.xsl; \
                      fi))
ALLXSLLFS := $(filter-out $(LFSTMP)/%, \
              $(shell if [ -d $(LFS_XML) ]; then \
                        find $(LFS_XML) -name \*.xsl; \
                      fi))

# Try to set the REV variable according to previous runs, except when
# set on the command line:
REV1 := $(shell if [ -f $(REVFILE) ] ; then cat $(REVFILE); fi)
ifneq ($(origin REV),command line)
  ifdef REV1
    REV = $(REV1)
  else
    REV = not defined
  endif
endif

ifneq ($(REV),sysv)
  ifneq ($(REV),systemd)
    $(error The REV variable is $(REV), but can only be 'sysv' or 'systemd')
  endif
endif

$(BOOK_XML): $(CONFIG_OUT)
	$(Q)$(TOPDIR)/gen_pkg_book.sh $(TRACKFILE) $(TOPDIR) $(BLFS_FULL) $(LFS_FULL)

$(CONFIG_OUT): $(CONFIG_CONFIG_IN)
	$(Q)CONFIG_="" KCONFIG_CONFIG=configuration \
	    python3 $(MENU)/menuconfig.py $(CONFIG_CONFIG_IN)

$(CONFIG_CONFIG_IN): $(PACK_LIST) $(XSLDIR)/gen_config.xsl
	$(Q)xsltproc --nonet -o $@ $(XSLDIR)/gen_config.xsl $(PACK_LIST)

$(PACK_LIST): $(XSLDIR)/gen_pkg_list.xsl $(XSLDIR)/specialCases.xsl $(TRACKFILE) $(LFS_FULL)
	$(Q)xsltproc --stringparam installed-packages $(TRACKFILE) \
	--stringparam lfs-full $(LFS_FULL) \
	-o $@.tmp $(XSLDIR)/gen_pkg_list.xsl $(BLFS_FULL)
	$(Q)xmllint --postvalid --format -o $@ $@.tmp
	$(Q)rm $@.tmp

# Beware of the echo '$(INITIAL_TRACK)' command below:
# if shell is bash or sh linked to bash, needs echo -e
# if shell is dash or sh linked to dash: echo is enough
# Don't ask me why
# So use /bin/echo (needs -e)
$(TRACKFILE): $(TRACKING_DIR)
	$(Q)if ! [ -f $@ ]; then \
	echo Initializing $(TRACKFILE) && \
	/bin/echo -e '$(INITIAL_TRACK)' > $@; \
	fi
	$(Q)for track in $(TRACKING_DIR)/*-*; do \
	if [ -f $$track ]; then \
	pack=$$(echo $$track | sed 's@.*/\(.*\)-[0-9c].*@\1@') && \
	version=$$(echo $$track | sed 's@.*-\([0-9c].*\)@\1@') && \
	xsltproc --stringparam packages $(PACK_LIST) \
	--stringparam package $$pack \
	--stringparam version $$version \
	-o track.tmp $(XSLDIR)/bump.xsl $@ && \
	sed -i 's@PACKDESC@$(TOPDIR)/packdesc.dtd@' track.tmp && \
	xmllint --format --postvalid track.tmp > $@; \
	fi; \
	rm -f $$track; \
	done; \
	rm -f track.tmp

$(TRACKING_DIR):
	@echo Creating $(TRACKING_DIR)
	$(Q)mkdir -p $@

$(XSLDIR)/specialCases.xsl: $(TOPDIR)/gen-special.sh $(BLFS_FULL)
	$(Q)$(TOPDIR)/gen-special.sh $(BLFS_FULL) $@

ifneq ($(REV),$(REV1))
$(BLFS_FULL): FORCE
$(LFS_FULL): FORCE
endif
$(LFS_FULL): $(LFS_XML) $(LFS_XML)/general.ent $(ALLXMLLFS) $(ALLXSLLFS)
	$(Q)cd $(LFS_XML) && if [ -x git-version.sh ] ; then \
	     echo "Processing LFS version..."; \
	     ./git-version.sh $(REV); \
	fi
	@echo "Processing LFS bootscripts..."
	$(Q)cd $(LFS_XML) && bash process-scripts.sh
	$(Q)[ -d $(LFSTMP) ] || mkdir -p $(LFSTMP)
	@echo "Adjusting LFS for revision $(REV)..."
	$(Q)xsltproc --nonet --xinclude                          \
	             --stringparam profile.revision $(REV)       \
	             --output $(LFSTMP)/lfs-prof.xml         \
	             $(LFS_XML)/stylesheets/lfs-xsl/profile.xsl \
	             $(LFS_XML)/index.xml
	@echo "Validating the LFS book..."
	$(Q)xmllint --nonet --noent --postvalid \
	  -o $@ $(LFSTMP)/lfs-prof.xml
	$(Q)rm -f $(LFS_XML)/appendices/*.script
	$(Q)cd $(LFS_XML) && ./aux-file-data.sh $@
	$(Q)echo $(REV) > $(REVFILE)

$(BLFS_FULL): $(BLFS_XML) $(BLFS_XML)/general.ent $(ALLXML) $(ALLXSL)
	$(Q)cd $(BLFS_XML) && if [ -x git-version.sh ] ; then \
	     echo "Processing BLFS version..."; \
	     ./git-version.sh $(REV); \
	fi
	$(Q)[ -d $(RENDERTMP) ] || mkdir -p $(RENDERTMP)
	@echo "Adjusting BLFS for revision $(REV)..."
	$(Q)xsltproc --nonet --xinclude                          \
	             --stringparam profile.revision $(REV)       \
	             --output $(RENDERTMP)/blfs-prof.xml         \
	             $(BLFS_XML)/stylesheets/lfs-xsl/profile.xsl \
	             $(BLFS_XML)/index.xml
	@echo "Validating the BLFS book..."
	$(Q)xmllint --nonet --noent --postvalid \
	  -o $@ $(RENDERTMP)/blfs-prof.xml
	$(Q)echo $(REV) > $(REVFILE)

all: update $(BOOK_XML)

# If in detached head state, "git pull" returns an error, so switch back to
# trunk before updating, and switch back to the branch/tag afterwards
update: $(BLFS_XML) $(LFS_XML)
	@echo Updating the book sources
ifneq (,$(LFS-BRANCH))
	$(Q)cd $(LFS_XML) && \
	git checkout trunk && \
	git pull && \
	if [ $(LFS-BRANCH) != trunk ]; then git checkout $(LFS-BRANCH); fi
endif
ifneq (,$(BLFS-BRANCH))
	$(Q)cd $(BLFS_XML) && \
        git checkout trunk && \
        git pull && \
        if [ $(BLFS-BRANCH) != trunk ]; then git checkout $(BLFS-BRANCH); fi
endif

$(LFS_XML):
	@echo Getting the LFS book sources...
	$(Q)git clone $(LFS-GIT) $@
ifneq (,$(LFS-BRANCH))
	if [ $(LFS-BRANCH) != trunk ]; then \
	    cd $@ && \
	    git checkout $(LFS-BRANCH); \
	fi
endif

$(BLFS_XML):
	@echo Getting the BLFS book sources...
	$(Q)git clone $(GIT) $@
ifneq (,$(BLFS-BRANCH))
	$(Q)if [ $(BLFS-BRANCH) != trunk ]; then \
	    cd $@ && \
	    git checkout $(BLFS-BRANCH); \
	fi
endif
# Clean up

clean:
	rm -f $(CONFIG_OUT) $(CONFIG_OUT).old $(TOPDIR)/packages.xml $(XSLDIR)/specialCases.xsl $(CONFIG_CONFIG_IN) book.xml
	rm -rf $(TOPDIR)/dependencies $(TOPDIR)/book-html $(TOPDIR)/scripts

FORCE:
.PHONY: clean all update $(CONFIG_OUT) version FORCE

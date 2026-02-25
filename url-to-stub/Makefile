SKILL_NAME := url-to-wiki-stub
SKILL_DIR  := skills/$(SKILL_NAME)
DIST_DIR   := dist
OUTPUT     := $(DIST_DIR)/$(SKILL_NAME).skill

SKILL_FILES := $(SKILL_DIR)/SKILL.md skills/wiki/SKILL.md skills/wikis/SKILL.md skills/reading-list/SKILL.md skills/question-log/SKILL.md

.PHONY: all clean

all: $(OUTPUT)

$(OUTPUT): $(SKILL_FILES)
	@mkdir -p $(DIST_DIR)
	@echo "Packaging $(SKILL_NAME)..."
	@cd skills && zip -r ../$(OUTPUT) $(SKILL_NAME)/ wiki/ wikis/ reading-list/ question-log/
	@echo "Built: $(OUTPUT)"

clean:
	rm -rf $(DIST_DIR)

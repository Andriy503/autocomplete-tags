<template>
    <div class="tags-input-root">
        <div :class="wrapperClass + ' tags-input'">
            <span class="tags-input-badge tags-input-badge-pill tags-input-badge-selected-default"
                v-for="(tag, index) in tags"
                :key="index"
            >
                <div class="content-tags">
                    <span v-html="tag.value"></span>
                    <i href="#" class="tags-input-remove" @click.prevent="removeTag(index)"></i>
                </div>
            </span>
            <input type="text"
                ref="taginput"
                v-model="input"
                @keydown.enter.prevent="tagFromInput(false)"
                @keydown.8="removeLastTag"
                @keydown.down="nextSearchResult"
                @keydown.up="prevSearchResult"
                @keydown="onKeyDown"
                @keyup="onKeyUp"
                @keyup.esc="clearSearchResults"
                @focus="onFocus"
                @blur="onBlur"
                @value="tags"
            >
            <input type="hidden" v-if="elementId"
                :name="elementId"
                :id="elementId"
                v-model="hiddenInput"
            >
        </div>

        <!-- Typeahead/Autocomplete -->
        <div v-show="searchResults.length" class="wrapper-typeahead-badges">
            <p v-if="typeaheadStyle === 'badges'" :class="`typeahead-${typeaheadStyle}`">
                <span v-for="(tag, index) in searchResults"
                    :key="index"
                    @mouseover="searchSelection = index"
                    @mousedown.prevent="tagFromSearchOnClick(tag)"
                    class="tags-input-badge"
                    v-bind:class="{
                        'tags-input-typeahead-item-default': index != searchSelection,
                        'tags-input-typeahead-item-highlighted-default': index == searchSelection
                    }">
                        <span class="autocomplete-value">{{ tag.value }}</span>
                    </span>
            </p>

            <ul v-else-if="typeaheadStyle === 'dropdown'" :class="`typeahead-${typeaheadStyle}`">
                <li v-for="(tag, index) in searchResults"
                :key="index"
                v-html="tag.text"
                @mouseover="searchSelection = index"
                @mousedown.prevent="tagFromSearchOnClick(tag)"
                v-bind:class="{
                    'tags-input-typeahead-item-default': index != searchSelection,
                    'tags-input-typeahead-item-highlighted-default': index == searchSelection
                }"></li>
            </ul>
        </div>
    </div>
</template>

<script>
export default {
    name: 'testTagsAndComplete',
    data() {
        return {
            badgeId: 0,
            wrapperClass: 'tags-input-wrapper-default',
            tags: [],
            input: '',
            oldInput: '',
            hiddenInput: '',
            searchResults: [],
            searchSelection: 0,
            selectedTag: -1,
            value: [],
            sortSearchResults: true,
            existingTags: [
                { key: 'children-of-bodom', value: 'Children of Bodom' },
                { key: 'epica', value: 'Epica' },
                { key: 'emperor', value: 'Emperor' },
                { key: 'shape-of-despair', value: 'Shape of Despair' },
                { key: 'protest-the-hero', value: 'Protest the Hero' },
                { key: 'my-dying-bride', value: 'My Dying Bride' },
                { key: 'ne-obliviscaris', value: 'Ne Obliviscaris' },
                { key: 'belakor', value: 'Be\'lakor' },
                { key: 'dark-lunacy', value: 'Dark Lunacy' },
                { key: 'dominia', value: 'Dominia' },
                { key: 'jason-becker', value: 'Jason Becker' },
                { key: 'jeff-loomis', value: 'Jeff Loomis' },
                { key: 'persefone', value: 'Persefone' }
            ],
            typeaheadActivationThreshold: 1,
            typeaheadMaxResults: 20,
            typeaheadStyle: 'badges',
            onlyExistingTags: false,
            caseSensitiveTags: false,
            deleteOnBackspace: true,
            allowDuplicates: false,
            addTagsOnComma: false,
            addTagsOnBlur: false,
            elementId: 'tags',
            typeahead: true,
            limit: 0,
            eventLog: ''
        };
    },

    created () {
        this.tagsFromValue();
        this.initialized();
    },

    watch: {
        tags() {
            console.log(this.tags)
            
            // Updating the hidden input
            this.hiddenInput = JSON.stringify(this.tags);

            // Update the bound v-model value
            this.$emit('input', this.tags);
        },

        value() {
            this.tagsFromValue();
        }
    },

    methods: {
        logEvent(text) {
            this.eventLog = text + this.eventLog;
        },
        onTagAdded (tag) {
            this.logEvent(`Tag added: "${JSON.stringify(tag)}"\n`);
        },
        initialized () {
            this.logEvent('Initialized\n');
        },
        onTagRemoved(tag) {
            this.logEvent(`Tag removed: "${JSON.stringify(tag)}"\n`);
        },
        onTagsUpdated() {
            this.logEvent('Tags updated\n');
        },
        // Remove reserved regex characters from a string so that they don't
        escapeRegExp(string) {
            return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        },

        // Add a tag whether from user input or from search results (typeahead)
        tagFromInput(ignoreSearchResults = false) {
            // If we're choosing a tag from the search results
            if (this.searchResults.length && this.searchSelection >= 0 && !ignoreSearchResults) {
                this.tagFromSearch(this.searchResults[this.searchSelection]);

                this.input = '';
            } else {
                // If we're adding an unexisting tag
                let text = this.input.trim();

                // If the new tag is not an empty string and passes validation
                if (!this.onlyExistingTags && text.length) {
                    this.input = '';

                    // Determine if the inputted tag exists in the existingTags
                    // array
                    let newTag = {
                        key: '',
                        value: text,
                    };

                    const searchQuery = this.escapeRegExp(
                        this.caseSensitiveTags
                            ? newTag.value
                            : newTag.value.toLowerCase()
                    );

                    for (let tag of this.existingTags) {
                        const compareable = this.caseSensitiveTags
                            ? tag.value
                            : tag.value.toLowerCase();

                        if (searchQuery == compareable) {
                            newTag = Object.assign({}, tag);

                            break;
                        }
                    }

                    this.addTag(newTag);
                }
            }
        },

        // Add a tag from search results when a user clicks on it
        tagFromSearchOnClick(tag) {
            this.tagFromSearch(tag);

            this.$refs['taginput'].blur();
        },

        // Add the selected tag from the search results.
        // Clear search results.
        // Clear user input.
        tagFromSearch(tag) {
            this.clearSearchResults();

            this.input = '';
            this.oldInput = '';

            this.addTag(tag);
        },

        // Add/Select a tag
        addTag(tag) {
            // Check if the limit has been reached
            if (this.limit > 0 && this.tags.length >= this.limit) {
                return false;
            }

            // Attach the tag if it hasn't been attached yet
            if (!this.tagSelected(tag)) {
                this.tags.push(tag);

                this.onTagAdded(tag)
                this.onTagsUpdated();
            }
        },

        // Remove the last tag in the tags array.
        removeLastTag() {
            if (!this.input.length && this.deleteOnBackspace) {
                this.removeTag(this.tags.length - 1);
            }
        },

        // Remove the selected tag at the specified index.
        removeTag(index) {
            let tag = this.tags[index];

            this.tags.splice(index, 1);

            this.onTagRemoved(tag)
            this.onTagsUpdated();
        },

        // Search the currently entered text in the list of existing tags
        searchTag() {
            if (this.typeahead !== true) {
                return false;
            }

            if (this.oldInput != this.input || (!this.searchResults.length && this.typeaheadActivationThreshold == 0)) {
                this.searchResults = [];
                this.searchSelection = 0;
                let input = this.input.trim();

                if ((input.length && input.length >= this.typeaheadActivationThreshold) || this.typeaheadActivationThreshold == 0) {
                    // Find all the existing tags which include the search text
                    const searchQuery = this.escapeRegExp(
                        this.caseSensitiveTags ? input : input.toLowerCase()
                    );

                    for (let tag of this.existingTags) {
                        const compareable = this.caseSensitiveTags
                            ? tag.value
                            : tag.value.toLowerCase();
  
                        if (compareable.search(searchQuery) > -1 && ! this.tagSelected(tag)) {
                            this.searchResults.push(tag);
                        }
                    }

                    // Sort the search results alphabetically
                    if (this.sortSearchResults) {
                        this.searchResults.sort((a, b) => {
                            if (a.value < b.value) return -1;
                            if (a.value > b.value) return 1;

                            return 0;
                        });
                    }

                    // Shorten Search results to desired length
                    if (this.typeaheadMaxResults > 0) {
                        this.searchResults = this.searchResults.slice(
                            0,
                            this.typeaheadMaxResults
                        );
                    }
                }

                this.oldInput = this.input;
            }
        },

        // Hide the typeahead if there's nothing intered into the input field.
        hideTypeahead() {
            if (! this.input.length) {
                this.$nextTick(() => {
                    this.clearSearchResults();
                });
            }
        },

        // Select the next search result in typeahead.
        nextSearchResult() {
            if (this.searchSelection + 1 <= this.searchResults.length - 1) {
                this.searchSelection++;
            }
        },

        // Select the previous search result in typeahead.
        prevSearchResult() {
            if (this.searchSelection > 0) {
                this.searchSelection--;
            }
        },

        // Clear/Empty the search results.
        clearSearchResults() {
            this.searchResults = [];
            this.searchSelection = 0;
        },

        // Clear the list of selected tags.
        clearTags() {
            this.tags.splice(0, this.tags.length);
        },

        // Replace the currently selected tags with the tags from the value.
        tagsFromValue() {
            if (this.value && this.value.length) {
                if (!Array.isArray(this.value)) {
                    console.error('Voerro Tags Input: the v-model value must be an array!');

                    return;
                }
                
                let tags = this.value;

                // Don't update if nothing has changed
                if (this.tags == tags) {
                    return;
                }

                this.clearTags();

                for (let tag of tags) {
                    this.addTag(tag);
                }
            } else {
                if (this.tags.length == 0) {
                    return;
                }

                this.clearTags();
            }
        },

        // Check if a tag is already selected.
        tagSelected(tag) {
            if (this.allowDuplicates) {
                return false;
            }

            if (! tag) {
                return false;
            }

            const searchQuery = this.escapeRegExp(
                this.caseSensitiveTags ? tag.value : tag.value.toLowerCase()
            );

            for (let selectedTag of this.tags) {
                const compareable = this.caseSensitiveTags
                    ? selectedTag.value
                    : selectedTag.value.toLowerCase();

                if (selectedTag.key == tag.key && compareable == searchQuery) {
                    return true;
                }
            }

            return false;
        },

        // Process all the keyup events.
        onKeyUp(e) {
            this.$emit('keyup', e);

            this.searchTag();
        },

        // Process all the keydown events.
        onKeyDown(e) {
            this.$emit('keydown', e);

            // Insert a new tag on comma keydown if the option is enabled
            if (e.key == ',') {
                if (this.addTagsOnComma) {
                    // The comma shouldn't actually be inserted
                    e.preventDefault();

                    // Add the inputed tag
                    this.tagFromInput(true);
                }
            }
        },

        // Process the onfocus event.
        onFocus(e) {
            this.$emit('focus', e)
            
            this.searchTag();
        },

        // Process the onblur event.
        onBlur(e) {
            this.$emit('blur', e)

            if (this.addTagsOnBlur) {
                // Add the inputed tag
                this.tagFromInput(true);
            }

            this.hideTypeahead();
        },
    }
}
</script>

<style>
.tags-input-root {
    position: relative;
    width: 420px;
    height: 40px;
    margin: 100px;
}

/* The input */
.tags-input {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
}

.tags-input input {
    flex: 1;
    background: transparent;
    border: none;
}

.tags-input input:focus {
    outline: none;
}

.tags-input input[type="text"] {
    color: #495057;
}

.tags-input-wrapper-default {
    padding: .5rem .25rem;

    background: #fff;

    border: 1px solid transparent;
    border-radius: .25rem;
    border-color: #dbdbdb;
}

/* The tag badges & the remove icon */
.tags-input span {
    margin-right: 0.3rem;
}

.tags-input-remove {
    cursor: pointer;
    position: relative;
    display: inline-block;
    width: 0.5rem;
    height: 0.5rem;
    overflow: hidden;
    background-color: white;
    padding: 5px;
    border-radius: 5px;
}

.tags-input-remove:before, .tags-input-remove:after {
    content: '';
    position: absolute;
    width: 100%;
    top: 50%;
    left: 0;
    background: #d9e3e9;
    
    height: 2px;
    margin-top: -1px;
}

.tags-input-remove:before {
    transform: rotate(45deg);
}
.tags-input-remove:after {
    transform: rotate(-45deg);
}

/* Tag badge styles */
.tags-input-badge {
    display: inline-block;
    padding: 0.25em 0.4em;
    font-size: 75%;
    font-weight: 700;
    line-height: 1;
    text-align: center;
    white-space: nowrap;
    vertical-align: baseline;
    border-radius: 0.25rem;
}

.tags-input-badge-selected-default {
    border-radius: 5px;
    background-color: #d9e3e9;
    height: 26px;
    margin-bottom: 5px;
}

.content-tags {
    margin-top: 3px;
}

.content-tags > span {
    font-size: 10px;
}

.wrapper-typeahead-badges {
    border: 1px solid #e9eef1;
    margin-top: 3px;
    border-radius: 5px;
    overflow-y: scroll;
    max-height: 200px;
}

/* Typeahead - badges */
.typeahead-badges > span {
    cursor: pointer;
    display: flex;
    width: 419px;
    height: 38px;
}

/* Typeahead - dropdown */
.typeahead-dropdown {
    list-style-type: none;
    padding: 0;
    margin: 0;
    position: absolute;
    width: 100%;
    z-index: 1000;
}

.typeahead-dropdown li {
    padding: .25rem 1rem;
    cursor: pointer;
}

/* Typeahead elements style/theme */
.tags-input-typeahead-item-default {
    color: #222350;
}

.tags-input-typeahead-item-highlighted-default {
    color: #222350;
    background-color: #e9eef1;
}

.autocomplete-value {
    margin: 10px;
}

</style>
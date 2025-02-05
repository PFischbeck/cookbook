<template>
    <div>
        <div class="kw">
            <transition-group
                v-if="uniqKeywords.length > 0"
                class="keywords"
                name="keyword-list"
                tag="ul"
            >
                <RecipeKeyword
                    v-for="keywordObj in selectedKeywords"
                    :key="keywordObj.name"
                    :name="keywordObj.name"
                    :count="keywordObj.count"
                    :title="t('cookbook', 'Toggle keyword')"
                    @keyword-clicked="keywordClicked(keywordObj)"
                    class="keyword active"
                />
                <RecipeKeyword
                    v-for="keywordObj in selectableKeywords"
                    :key="keywordObj.name"
                    :name="keywordObj.name"
                    :count="keywordObj.count"
                    :title="t('cookbook', 'Toggle keyword')"
                    @keyword-clicked="keywordClicked(keywordObj)"
                    class="keyword"
                />
                <RecipeKeyword
                    v-for="keywordObj in unavailableKeywords"
                    :key="keywordObj.name"
                    :name="keywordObj.name"
                    :count="keywordObj.count"
                    :title="
                        // prettier-ignore
                        t('cookbook','Keyword not contained in visible recipes')
                    "
                    @keyword-clicked="keywordClicked(keywordObj)"
                    class="keyword disabled"
                />
            </transition-group>
        </div>
        <ul class="recipes">
            <li
                v-for="recipeObj in recipeObjects"
                :key="recipeObj.recipe.recipe_id"
                v-show="recipeObj.show"
            >
                <router-link :to="'/recipe/' + recipeObj.recipe.recipe_id">
                    <lazy-picture
                        v-if="recipeObj.recipe.imageUrl"
                        class="recipe-thumbnail"
                        :lazy-src="recipeObj.recipe.imageUrl"
                        :blurred-preview-src="
                            recipeObj.recipe.imagePlaceholderUrl
                        "
                        :width="105"
                        :height="105"
                    />
                    <span>{{ recipeObj.recipe.name }}</span>
                </router-link>
            </li>
        </ul>
    </div>
</template>

<script>
import LazyPicture from "./LazyPicture"
import RecipeKeyword from "./RecipeKeyword"

export default {
    name: "RecipeList",
    props: ["recipes"],
    components: {
        LazyPicture,
        RecipeKeyword,
    },
    data() {
        return {
            // String-based filters applied to the list
            filters: "",
            // All keywords to filter the recipes for (conjunctively)
            keywordFilter: [],
        }
    },
    computed: {
        /**
         * An array of all keywords in all recipes. These are neither sorted nor unique
         */
        rawKeywords() {
            var keywordArray = this.recipes.map(function (r) {
                if (!"keywords" in r) {
                    return []
                }
                if (r.keywords != null) {
                    return r.keywords.split(",")
                } else {
                    return []
                }
            })
            return [].concat(...keywordArray)
        },
        /**
         * An array of sorted and unique keywords over all the recipes
         */
        uniqKeywords() {
            function uniqFilter(value, index, self) {
                return self.indexOf(value) === index
            }
            return this.rawKeywords.sort().filter(uniqFilter)
        },
        /**
         * An array of objects that contain the keywords plus a count of recipes asociated with these keywords
         */
        keywordsWithCount() {
            let $this = this
            return this.uniqKeywords
                .map(function (kw) {
                    return {
                        name: kw,
                        count: $this.rawKeywords.filter((kw2) => kw === kw2)
                            .length,
                    }
                })
                .sort(function (k1, k2) {
                    if (k1.count != k2.count) {
                        // Distinguish by number
                        return k2.count - k1.count
                    }
                    // Distinguish by keyword name
                    return k1.name.toLowerCase() > k2.name.toLowerCase()
                        ? 1
                        : -1
                })
        },
        /**
         * An array of keyword objects that are currently in use for filtering
         */
        selectedKeywords() {
            return this.keywordsWithCount.filter((kw) =>
                this.keywordFilter.includes(kw.name)
            )
        },
        /**
         * An array of those keyword objects that are currently not in use for filtering
         */
        unselectedKeywords() {
            return this.keywordsWithCount.filter(
                (kw) => !this.selectedKeywords.includes(kw)
            )
        },
        /**
         * An array of all recipes that are part in all filtered keywords
         */
        recipesFilteredByKeywords() {
            let $this = this
            return this.recipes.filter(function (r) {
                if ($this.keywordFilter.length == 0) {
                    // No filtering, keep all
                    return true
                }

                if (r.keywords === null) {
                    return false
                }

                function keywordInRecipePresent(kw, r) {
                    if (!r.keywords) {
                        return false
                    }

                    let keywords = r.keywords.split(",")
                    return keywords.includes(kw.name)
                }

                return $this.selectedKeywords
                    .map((kw) => keywordInRecipePresent(kw, r))
                    .reduce((l, r) => l && r)
            })
        },
        /**
         * An array of the finally filtered recipes, that is both filtered for keywords as well as string-based name filtering
         */
        filteredRecipes() {
            let ret = this.recipesFilteredByKeywords
            let $this = this

            if (this.filters) {
                ret = ret.filter(function (r) {
                    return r.name
                        .toLowerCase()
                        .includes($this.filters.toLowerCase())
                })
            }

            return ret
        },
        /**
         * An array of keywords that are yet unselected but some visible recipes are associated
         */
        selectableKeywords() {
            if (this.unselectedKeywords.length === 0) {
                return []
            }

            let $this = this
            return this.unselectedKeywords.filter(function (kw) {
                return $this.filteredRecipes
                    .map(function (r) {
                        return (
                            r.keywords &&
                            r.keywords.split(",").includes(kw.name)
                        )
                    })
                    .reduce((l, r) => l || r, false)
            })
        },
        /**
         * An array of known keywords that are not associated with any visible recipe
         */
        unavailableKeywords() {
            return this.unselectedKeywords.filter(
                (kw) => !this.selectableKeywords.includes(kw)
            )
        },
        // An array of recipe objects of all recipes with links to the recipes and a property if the recipe is to be shown
        recipeObjects() {
            return this.recipes.map(function (r) {
                return {
                    recipe: r,
                    show: this.filteredRecipes.includes(r),
                }
            }, this)
        },
    },
    methods: {
        /**
         * Callback for click on keyword, add to or remove from list
         */
        keywordClicked: function (keyword) {
            const index = this.keywordFilter.indexOf(keyword.name)
            if (index > -1) {
                this.keywordFilter.splice(index, 1)
            } else {
                this.keywordFilter.push(keyword.name)
            }
        },
    },
    mounted() {
        this.$root.$off("applyRecipeFilter")
        this.$root.$on("applyRecipeFilter", (value) => {
            this.filters = value
        })
    },
}
</script>

<style scoped>
.kw {
    width: 100%;
    max-height: 6.7em;
    padding: 0.1em;
    margin-bottom: 1em;
    overflow-x: hidden;
    overflow-y: scroll;
}

.keywords {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    padding: 0.5rem 1rem;
}

.keyword {
    display: inline-block;
}

.keyword-list-move {
    transition: transform 0.5s;
}

.recipes {
    display: flex;
    width: 100%;
    flex-direction: row;
    flex-wrap: wrap;
}

.recipes li {
    width: 300px;
    max-width: 100%;
    margin: 0.5rem 1rem 1rem;
}
.recipes li a {
    display: block;
    height: 105px;
    border-radius: 3px;
    box-shadow: 0 0 3px #aaa;
}
.recipes li a:hover {
    box-shadow: 0 0 5px #888;
}

.recipes li .recipe-thumbnail {
    position: relative;
    overflow: hidden;
    width: 105px;
    height: 105px;
    background-color: #bebdbd;
    border-radius: 3px 0 0 3px;
    float: left;
}

.recipes li span {
    display: block;
    padding: 0.5rem 0.5em 0.5rem calc(105px + 0.5rem);
}
</style>

<template>
    <div>
        <div
            v-if="actions.length > 0 || availablePivotActions.length > 0"
            class="block md:flex md:custom_align_right  items-center"
        >
                    <button
                        v-for="action in actions"
                        :key="action.uriKey"
                        @click.prevent="determineActionStrategy(action.uriKey)"
                        class="btn md:mb-2 flex btn-default hover:bg-primary hover:text-white  btn-dark bg-primary text-white sm:flex-none lg:flex items-center justify-center px-3 mr-2 border-primary" >
                        <span class="span-icon" v-html="action.icon"></span>
                        <span class="text-base  whitespace-no-wrap">{{ action.name }} test</span>
                    </button>

        </div>

        <!-- Action Confirmation Modal -->
        <!-- <portal to="modals"> -->
        <transition name="fade">
            <component
                style="margin-top:50px;"
                :is="selectedAction.component"
                :working="working"
                v-if="confirmActionModalOpened"
                :selected-resources="selectedResources"
                :resource-name="resourceName"
                :action="selectedAction"
                :errors="errors"
                @confirm="executeAction"
                @close="confirmActionModalOpened = false"
            />
        </transition>
        <!-- </portal> -->
    </div>
</template>

<script>
    import _ from 'lodash'
    import { Errors, InteractsWithResourceInformation } from 'laravel-nova'

    export default {
        mixins: [InteractsWithResourceInformation],

        props: {
            selectedResources: {
                type: [Array, String],
                default: () => [],
            },
            resourceName: String,
            actions: {},
            pivotActions: {},
            pivotName: String,
            endpoint: {
                type: String,
                default: null,
            },
            queryString: {
                type: Object,
                default: () => ({
                    currentSearch: '',
                    encodedFilters: '',
                    currentTrashed: '',
                    viaResource: '',
                    viaResourceId: '',
                    viaRelationship: '',
                }),
            },
        },

        data: () => ({
            working: false,
            errors: new Errors(),
            selectedActionKey: '',
            confirmActionModalOpened: false,
        }),

        watch: {
            /**
             * Watch the actions property for changes.
             */
            actions() {
                this.selectedActionKey = ''
                this.initializeActionFields()
            },

            /**
             * Watch the pivot actions property for changes.
             */
            pivotActions() {
                this.selectedActionKey = ''
                this.initializeActionFields()
            },
        },

        methods: {
            /**
             * Determine whether the action should redirect or open a confirmation modal
             */
            determineActionStrategy(key) {
                this.selectedActionKey=key;
                if (this.selectedAction.withoutConfirmation) {
                    this.executeAction()
                } else {
                    this.openConfirmationModal()
                }
            },

            /**
             * Confirm with the user that they actually want to run the selected action.
             */
            openConfirmationModal() {
                this.confirmActionModalOpened = true
            },

            /**
             * Close the action confirmation modal.
             */
            closeConfirmationModal() {
                this.confirmActionModalOpened = false
            },

            /**
             * Initialize all of the action fields to empty strings.
             */
            initializeActionFields() {
                _(this.allActions).each(action => {
                    _(action.fields).each(field => {
                        field.fill = () => ''
                    })
                })
            },

            /**
             * Execute the selected action.
             */
            executeAction() {
                this.working = true

                if (this.selectedResources.length == 0) {
                    alert(this.__('Please select a resource to perform this action on.'))
                    return
                }

                Nova.request({
                    method: 'post',
                    url: this.endpoint || `/nova-api/${this.resourceName}/action`,
                    params: this.actionRequestQueryString,
                    data: this.actionFormData(),
                })
                    .then(response => {
                        this.confirmActionModalOpened = false
                        this.handleActionResponse(response.data)
                        this.working = false
                    })
                    .catch(error => {
                        this.working = false

                        if (error.response.status == 422) {
                            this.errors = new Errors(error.response.data.errors)
                        }
                    })
            },

            /**
             * Gather the action FormData for the given action.
             */
            actionFormData() {
                return _.tap(new FormData(), formData => {
                    formData.append('resources', this.selectedResources)

                    _.each(this.selectedAction.fields, field => {
                        field.fill(formData)
                    })
                })
            },

            /**
             * Handle the action response. Typically either a message, download or a redirect.
             */
            handleActionResponse(response) {
                if (response.message) {
                    this.$emit('actionExecuted')
                    this.$toasted.show(response.message, { type: 'success' })
                } else if (response.deleted) {
                    this.$emit('actionExecuted')
                } else if (response.danger) {
                    this.$emit('actionExecuted')
                    this.$toasted.show(response.danger, { type: 'error' })
                } else if (response.download) {
                    let link = document.createElement('a')
                    link.href = response.download
                    link.download = response.name
                    document.body.appendChild(link)
                    link.click()
                    document.body.removeChild(link)
                } else if (response.redirect) {
                    window.location = response.redirect
                } else if (response.openInNewTab) {
                    window.open(response.openInNewTab, '_blank')
                } else {
                    this.$emit('actionExecuted')
                    this.$toasted.show(this.__('The action ran successfully!'), { type: 'success' })
                }
            },
        },

        computed: {
            selectedAction() {
                if (this.selectedActionKey) {
                    return _.find(this.allActions, a => a.uriKey == this.selectedActionKey)
                }
            },

            /**
             * Get the query string for an action request.
             */
            actionRequestQueryString() {
                return {
                    action: this.selectedActionKey,
                    pivotAction: this.selectedActionIsPivotAction,
                    search: this.queryString.currentSearch,
                    filters: this.queryString.encodedFilters,
                    trashed: this.queryString.currentTrashed,
                    viaResource: this.queryString.viaResource,
                    viaResourceId: this.queryString.viaResourceId,
                    viaRelationship: this.queryString.viaRelationship,
                }
            },

            /**
             * Determine if the selected action is a pivot action.
             */
            selectedActionIsPivotAction() {
                return (
                    this.hasPivotActions &&
                    Boolean(_.find(this.pivotActions.actions, a => a === this.selectedAction))
                )
            },

            /**
             * Get all of the available actions.
             */
            allActions() {

                return this.actions.concat(this.pivotActions.actions)
            },

            /**
             * Get all of the available non-pivot actions for the resource.
             */
            availableActions() {
                return _(this.actions)
                    .filter(action => {
                        if (this.selectedResources != 'all') {
                            return true
                        }

                        return action.availableForEntireResource
                    })
                    .value()
            },

            /**
             * Determine whether there are any pivot actions
             */
            hasPivotActions() {
                return this.availablePivotActions.length > 0
            },

            /**
             * Get all of the available pivot actions for the resource.
             */
            availablePivotActions() {

                return _(this.pivotActions.actions)
                    .filter(action => {
                        if (this.selectedResources != 'all') {
                            return true
                        }

                        return action.availableForEntireResource
                    })
                    .value()
            },
        },
    }
</script>
<style lang="scss">
    .span-icon {
        margin-right: .75rem;
        width: 1.25rem;
        height: 1.25rem;
    }
    .span-icon * {
        display:block;
    }
    .btn:hover *, .btn:hover * path {
        fill: var(--white);
    }
    .border-primary {
        border: 1px solid var(--primary);
    }
    .btn:hover .icon_hover,.btn:hover .icon_hover path {
        fill:var(--white);
    }
    @media (min-width:768px) {
        .md\:flex{
            display:flex;
        }
    }
    @media(max-width:767px) {
        .md\:mb-2{
            margin-bottom:.5rem
        }
       .md\:custom_align_right {
           display: flex;
           flex-direction: column;
           justify-content: flex-end;
           align-items: flex-end;
       }
    }
</style>

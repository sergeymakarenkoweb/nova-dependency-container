<template>
	<div v-if="dependenciesSatisfied">
		<div v-for="childField in field.fields">
			<component
				:is="'form-' + childField.component"
				:errors="errors"
				:resource-id="resourceId"
				:resource-name="resourceName"
				:field="childField"
				:ref="'field-' + childField.attribute"
			/>
		</div>
	</div>
</template>

<script>
	import {FormField, HandlesValidationErrors} from 'laravel-nova'

	export default {
		mixins: [FormField, HandlesValidationErrors],

		props: ['resourceName', 'resourceId', 'field'],

		mounted() {
			this.registerDependencyWatchers(this.$root, function() {
				this.updateDependencyStatus();
			});
		},

		data() {
			return {
				dependencyValues: {},
				dependenciesSatisfied: false,
			}
		},

		methods: {

			// @todo: refactor entire watcher procedure, this approach isn't maintainable ..
			registerDependencyWatchers(root, callback) {
				callback = callback || null;
				root.$children.forEach(component => {
					if (this.componentIsDependency(component)) {

						// @todo: change `findWatchableComponentAttribute` to return initial state(s) of current dependency.
						let attribute = this.findWatchableComponentAttribute(component),
							initial_value = this.getInitValue(component); // @note: quick-fix for issue #88
                        let self = this;
						component.$watch(attribute, function (value) {
						    if (value === undefined) {
						        return
                            }
						    value = self.getValue(this, value, attribute)
							self.dependencyValues[component.field.attribute] = value;
							// @todo: change value as argument for `updateDependencyStatus`
							self.updateDependencyStatus()
						}, {immediate: true});
						// @todo: move to initial state
						// @note quick-fix for issue #88
						if (attribute === 'fieldTypeName') {
							initial_value = component.field.resourceLabel;
						}

						// @todo: replace with `updateDependencyStatus(initial_value)` and let it resolve dependency state
						this.dependencyValues[component.field.attribute] = initial_value;
					}

					this.registerDependencyWatchers(component)
				});

				if (callback !== null) {
					callback.call(this);
				}
			},
            getValue(component, value, attribute = 'value') {
			    if (component.field.component !== 'nova-belongsto-depend') {
			        if (attribute !== 'selectedResource') {
			            return value;
                    }
			        return (value && value.value) || null
                } else {
			        let dependency = this.field.dependencies.find((dep) => {
			            return dep.field === component.field.attribute
                    })
                    return !Array.isArray(value) && typeof value === 'object'
                        ? value[dependency['property']]
                        : null
                }
            },
            getInitValue(component) {
			    if (component.field.component !== 'nova-belongsto-depend') {
			        return component.field.value
                } else {
			        let value = component.field.options.find(function (option) {
			            return option[component.field.modelPrimaryKey] === component.field.valueKey
                    })
                    return this.getValue(component, value, 'value')
                }
            },

			// @todo: not maintainable, move to factory
			findWatchableComponentAttribute(component) {
				let attribute;
				switch(component.field.component) {
					case 'belongs-to-many-field':
					case 'belongs-to-field':
						attribute = 'selectedResource';
						break;
					case 'morph-to-field':
						attribute = 'fieldTypeName';
						break;
					default:
						attribute = 'value';
				}
				return attribute;
			},

			componentIsDependency(component) {
				if (component.field === undefined) {
					return false;
				}

				for (let dependency of this.field.dependencies) {
					// #93 compatability with flexible-content, which adds a generated attribute for each field
					if (component.field.attribute === (this.field.attribute + dependency.field)) {
						return true;
					}
				}

				return false;
			},

			// @todo: align this method with the responsibility of updating the dependency, not verifying the dependency "values"
			updateDependencyStatus() {
				for (let dependency of this.field.dependencies) {

					// #93 compatability with flexible-content, which adds a generated attribute for each field
					let dependencyValue = this.dependencyValues[(this.field.attribute + dependency.field)];
					if (dependency.hasOwnProperty('empty') && !dependencyValue) {
						this.dependenciesSatisfied = true;
						return;
					}

					if (dependency.hasOwnProperty('notEmpty') && dependencyValue) {
						this.dependenciesSatisfied = true;
						return;
					}

					if (dependency.hasOwnProperty('nullOrZero') && 1 < [undefined, null, 0, '0'].indexOf(dependencyValue) ) {
						this.dependenciesSatisfied = true;
						return;
					}

					if (dependency.hasOwnProperty('not') && dependencyValue !== dependency.not) {
						this.dependenciesSatisfied = true;
						return;
					}

					if (dependency.hasOwnProperty('value') && dependencyValue == dependency.value) {
						this.dependenciesSatisfied = true;
						return;
					}
				}

				this.dependenciesSatisfied = false;
			},

			fill(formData) {
				if (this.dependenciesSatisfied) {
					_.each(this.field.fields, field => {
						if (field.fill) {
							field.fill(formData)
						}
					})
				}
			}

		}
	}
</script>

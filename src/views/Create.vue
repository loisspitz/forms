<!--
  - @copyright Copyright (c) 2018 René Gieling <github@dartcafe.de>
  -
  - @author René Gieling <github@dartcafe.de>
  - @author Nick Gallo
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <http://www.gnu.org/licenses/>.
  -
  -
  - UPDATE: Adds Quiz option and takes the input:
  - is yet to store input of quizzes and cannot represtent them
  - requires quizFormItem.vue (should be added to svn)
  -->

<template>
	<AppContent>
		<!-- Show results & sidebar button -->
		<TopBar>
			<button class="primary" @click="showResults">
				<span class="icon-forms-white" role="img" />
				{{ t('forms', 'Show results') }}
			</button>
			<button v-tooltip="t('forms', 'Toggle settings')" @click="toggleSidebar">
				<span class="icon-settings" role="img" />
			</button>
		</TopBar>

		<!-- Forms title & description-->
		<header>
			<label class="hidden-visually" for="form-title">{{ t('forms', 'Title') }}</label>
			<input
				id="form-title"
				v-model="form.event.title"
				:minlength="0"
				:placeholder="t('forms', 'Title')"
				:required="true"
				autofocus
				type="text"
				@click="selectIfUnchanged">
			<label class="hidden-visually" for="form-desc">{{ t('forms', 'Description') }}</label>
			<textarea
				id="form-desc"
				ref="description"
				v-model="form.event.description"
				:placeholder="t('forms', 'Description')"
				@keydown="autoSizeDescription" />
		</header>

		<section>
			<!-- Add new questions toolbar -->
			<div class="question-toolbar" role="toolbar">
				<Actions ref="questionMenu"
					v-tooltip="t('forms', 'Add a question to this form')"
					:aria-label="t('forms', 'Add a question to this form')"
					:open.sync="questionMenuOpened"
					default-icon="icon-add-white">
					<ActionButton v-for="type in answerTypes"
						:key="type.label"
						class="question-toolbar__question"
						:icon="type.icon"
						@click="addQuestion">
						{{ type.label }}
					</ActionButton>
				</Actions>
			</div>

			<!-- <div id="quiz-form-selector-text">
				<label for="ans-type">Answer Type: </label>
				<select v-model="selected">
					<option value="" disabled>
						Select
					</option>
					<option v-for="option in answerTypes" :key="option.value" :value="option.value">
						{{ option.text }}
					</option>
				</select>
				<input v-model="newQuizQuestion" :placeholder="t('forms', 'Add Question')" @keyup.enter="addQuestion()">
				<button id="questButton"
					@click="addQuestion()">
					{{ t('forms', 'Add Question') }}
				</button>
			</div> -->

			<!-- No questions -->
			<EmptyContent v-if="form.options.formQuizQuestions.length === 0">
				{{ t('forms', 'This form does not have any questions') }}
				<template #desc>
					<button class="empty-content__button primary" @click="openQuestionMenu">
						<span class="icon-add-white" />
						{{ t('forms', 'Add a new one') }}
					</button>
				</template>
			</EmptyContent>

			<!-- Questions list -->
			<transitionGroup
				v-else
				id="form-list"
				name="list"
				tag="ul"
				class="form-table">
				<li
					is="quiz-form-item"
					v-for="(question, index) in form.options.formQuizQuestions"
					:key="question.id"
					:question="question"
					:type="question.type"
					@add-answer="addAnswer"
					@remove-answer="removeAnswer"
					@remove="form.options.formQuizQuestions.splice(index, 1)" />
			</transitionGroup>
		</section>
	</AppContent>
</template>

<script>
import { emit } from '@nextcloud/event-bus'
import { showError } from '@nextcloud/dialogs'
import axios from '@nextcloud/axios'
import debounce from 'debounce'

import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import Actions from '@nextcloud/vue/dist/Components/Actions'
import AppContent from '@nextcloud/vue/dist/Components/AppContent'

import answerTypes from '../models/AnswerTypes'
import EmptyContent from '../components/EmptyContent'
import QuizFormItem from '../components/quizFormItem'
import TopBar from '../components/TopBar'
import ViewsMixin from '../mixins/ViewsMixin'

export default {
	name: 'Create',
	components: {
		ActionButton,
		Actions,
		AppContent,
		EmptyContent,
		QuizFormItem,
		TopBar,
	},

	mixins: [ViewsMixin],

	data() {
		return {
			questionMenuOpened: false,
			newQuizAnswer: '',
			newQuizQuestion: '',
			nextQuizAnswerId: 1,
			nextQuizQuestionId: 1,
			writingForm: false,
			selected: '',
			uniqueName: false,
			uniqueAnsName: true,
			answerTypes,
		}
	},

	computed: {
		title() {
			if (this.form.event.title === '') {
				return t('forms', 'Create new form')
			} else {
				return this.form.event.title

			}
		},

		saveButtonTitle() {
			if (this.writingForm) {
				return t('forms', 'Writing form')
			} else if (this.form.mode === 'edit') {
				return t('forms', 'Update form')
			} else {
				return t('forms', 'Done')
			}
		},

	},

	watch: {
		title() {
			// only used when the title changes after page load
			document.title = t('forms', 'Forms') + ' - ' + this.title
		},

		form: {
			deep: true,
			handler: function() {
				this.debounceWriteForm()
			},
		},
	},

	created() {
		if (this.$route.name === 'edit') {
			this.form.mode = 'edit'
		} else if (this.$route.name === 'clone') {
			// TODO: CLONE
		}
	},

	methods: {

		switchSidebar() {
			this.sidebar = !this.sidebar
		},

		checkNames() {
			this.uniqueName = true
			this.form.options.formQuizQuestions.forEach(q => {
				if (q.text === this.newQuizQuestion) {
					this.uniqueName = false
				}
			})
		},

		addQuestion() {
			this.checkNames()
			if (this.selected === '') {
				showError(t('forms', 'Select a question type!'), { duration: 3000 })
			} else if (!this.uniqueName) {
				showError(t('forms', 'Cannot have the same question!'))
			} else {
				if (this.newQuizQuestion !== null & this.newQuizQuestion !== '' & (/\S/.test(this.newQuizQuestion))) {
					this.form.options.formQuizQuestions.push({
						id: this.nextQuizQuestionId++,
						text: this.newQuizQuestion,
						type: this.selected,
						answers: [],
					})
				}
				this.newQuizQuestion = ''
			}
		},

		checkAnsNames(item, question) {
			this.uniqueAnsName = true
			question.answers.forEach(q => {
				if (q.text === item.newQuizAnswer) {
					this.uniqueAnsName = false
				}
			})
		},

		removeAnswer(item, question, index) {
			item.formQuizAnswers.splice(index, 1)
			question.answers.splice(index, 1)
		},

		addAnswer(item, question) {
			this.checkAnsNames(item, question)
			if (!this.uniqueAnsName) {
				showError(t('forms', 'Two answers cannot be the same!'), { duration: 3000 })
			} else {
				if (item.newQuizAnswer !== null & item.newQuizAnswer !== '' & (/\S/.test(item.newQuizAnswer))) {
					item.formQuizAnswers.push({
						id: item.nextQuizAnswerId,
						text: item.newQuizAnswer,
					})
					question.answers.push({
						id: item.nextQuizAnswerId,
						text: item.newQuizAnswer,
					})
					item.nextQuizAnswerId++
				}
				item.newQuizAnswer = ''
			}
		},

		allHaveAns() {
			let haveAns = true
			this.form.options.formQuizQuestions.forEach(q => {
				if (q.type !== 'text' && q.type !== 'comment' && q.answers.length === 0) {
					haveAns = false
				}
			})
			return haveAns
		},

		autoSizeDescription() {
			const textarea = this.$refs.description
			textarea.style.cssText = 'height:auto; padding:0'
			textarea.style.cssText = `height: ${textarea.scrollHeight + 20}px`
		},

		debounceWriteForm: debounce(function() {
			this.writeForm()
		}, 200),

		async writeForm() {
			if (this.form.event.title.length === 0 | !(/\S/.test(this.form.event.title))) {
				showError(t('forms', 'Title must not be empty!'))
			} else if (!this.allHaveAns()) {
				showError(t('forms', 'All questions need answers!'))
			} else if (this.form.event.expiration & this.form.event.expirationDate === '') {
				showError(t('forms', 'Need to pick an expiration date!'))
			} else {
				this.writingForm = true
				try {
					await axios.post(OC.generateUrl('apps/forms/write/form'), this.form)
					this.form.mode = 'edit'
					this.writingForm = false
					console.debug(t('forms', '%n successfully saved', 1, this.form))
				} catch (error) {
					this.writingForm = false
					showError(t('forms', 'Error while saving form'))
					console.error(error)
				}
			}
		},

		/**
		 * Topbar methods
		 */
		showResults() {
			this.$router.push({
				name: 'results',
				params: {
					hash: this.form.event.hash,
				},
			})
		},
		toggleSidebar() {
			emit('toggleSidebar')
		},

		/**
		 * Add question methods
		 */
		openQuestionMenu() {
			// TODO: fix the vue components to allow external click triggers without
			// conflicting with the click outside directive
			setTimeout(() => {
				this.questionMenuOpened = true
			}, 100)
		},

		/**
		 * Select the text in the input if it is still set to 'New form'
		 * @param {Event} e the click event
		 */
		selectIfUnchanged(e) {
			if (e.target && e.target.value === t('forms', 'New form')) {
				e.target.select()
			}
		},
	},
}
</script>

<style lang="scss">
#app-content {
	display: flex;
	align-items: center;
	flex-direction: column;

	header,
	section {
		width: 100%;
		max-width: 900px;
	}

	// Title & description header
	header {
		display: flex;
		flex-direction: column;
		margin: 44px;

		#form-title,
		#form-desc {
			width: 100%;
			margin: 10px; // aerate the header
			padding: 0; // makes alignment and desc height calc easier
			border: none;
		}
		#form-title {
			font-size: 2em;
		}
		#form-desc {
			min-height: 60px;
			max-height: 200px;
			padding-left: 2px; // align with title (compensate font size diff)
			resize: none;
		}
	}

	.empty-content__button {
		margin: 5px;
		> span {
			margin-right: 5px;
			cursor: pointer;
			opacity: 1;
		}
	}

	// Questions container
	section {
		position: relative;
		display: flex;
		flex-direction: column;
		min-height: 200vh;

		.question-toolbar {
			position: sticky;
			z-index: 50;
			top: var(--header-height);
			display: flex;
			align-items: center;
			align-self: flex-end;
			width: 44px;
			height: var(--top-bar-height);
			// make sure this doesn't take any space and appear floating
			margin-top: -44px;
			.icon-add-white {
				opacity: 1;
				border-radius: 50%;
				// TODO: standardize on components
				background-color: var(--color-primary-element);
				&:hover,
				&:focus,
				&:active {
					background-color: var(--color-primary-element-light) !important;
				}
			}
		}
	}
}

/* Transitions for inserting and removing list items */
.list-enter-active,
.list-leave-active {
	transition: all .5s ease;
}

.list-enter,
.list-leave-to {
	opacity: 0;
}

.list-move {
	transition: transform .5s;
}

#form-item-selector-text {
	> input {
		width: 100%;
	}
}

.form-table {
	> li {
		display: flex;
		overflow: hidden;
		align-items: baseline;
		min-height: 24px;
		padding-right: 8px;
		padding-left: 8px;
		white-space: nowrap;
		border-bottom: 1px solid var(--color-border);
		line-height: 24px;

		&:active,
		&:hover {
			transition: var(--background-dark) .3s ease;
			background-color: var(--color-background-dark); //$hover-color;
		}

		> div {
			display: flex;
			flex-grow: 1;
			padding-right: 4px;
			white-space: normal;
			opacity: .7;
			font-size: 1.2em;
			&.avatar {
				flex-grow: 0;
			}
		}

		> div:nth-last-child(1) {
			flex-grow: 0;
			flex-shrink: 0;
			justify-content: center;
		}
	}
}

button {
	&.button-inline {
		border: 0;
		background-color: transparent;
	}
}

.tab {
	display: flex;
	flex-wrap: wrap;
}
.selectUnit {
	display: flex;
	align-items: center;
	flex-wrap: nowrap;
	> label {
		padding-right: 4px;
	}
}

#shiftDates {
	min-width: 16px;
	min-height: 16px;
	margin: 0;
	padding: 10px;
	padding-left: 34px;
	text-align: left;
	background-repeat: no-repeat;
	background-position: 10px center;
}

</style>

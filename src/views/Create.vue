<!--
  - @copyright Copyright (c) 2018 René Gieling <github@dartcafe.de>
  -
  - @author René Gieling <github@dartcafe.de>
  - @author Nick Gallo
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
				type="text">
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
			<!-- <div class="question-toolbar" role="toolbar">
				<button v-for="type in answerTypes"
					:key="type.label"
					class="question-toolbar__question"
					@click="addQuestion">
					<span class="question-toolbar__icon" :class="type.icon" />
					{{ type.label }}
				</button>
			</div> -->

			<div id="quiz-form-selector-text">
				<!--shows inputs for question types: drop down box to select the type, text box for question, and button to add-->
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
			</div>

			<!-- No questions -->
			<EmptyContent v-if="form.options.formQuizQuestions.length === 0">
				{{ t('forms', 'This form does not have any questions') }}
				<template #desc>
					<button class="primary" @click="openQuestionMenu">
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
import { generateUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import moment from '@nextcloud/moment'
import { emit } from '@nextcloud/event-bus'
import { showError } from '@nextcloud/dialogs'
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
			this.$refs.questionMenu.opened = true
		},
	},
}
</script>

<style lang="scss">
#app-content {
	display: flex;
	flex-direction: column;
	align-items: center;
	header,
	section {
		width: 100%;
		max-width: 900px;
	}

	header {
		display: flex;
		flex-direction: column;
		margin: 44px;

		#form-title,
		#form-desc {
			width: 100%;
			border: none;
			margin: 10px; // aerate the header
			padding: 0; // makes alignment and desc height calc easier
		}
		#form-title {
			font-size: 2em;
		}
		#form-desc {
			min-height: 60px;
			max-height: 200px;
			padding-left: 2px; // align with title (compensate font size diff)
			resize: none
		}
	}

	section {
		position: relative;
	}
}

/* Transitions for inserting and removing list items */
.list-enter-active,
.list-leave-active {
	transition: all 0.5s ease;
}

.list-enter,
.list-leave-to {
	opacity: 0;
}

.list-move {
	transition: transform 0.5s;
}
/*  */

#form-item-selector-text {
	> input {
		width: 100%;
	}
}

.form-table {
	> li {
		display: flex;
		align-items: baseline;
		padding-left: 8px;
		padding-right: 8px;
		line-height: 24px;
		min-height: 24px;
		border-bottom: 1px solid var(--color-border);
		overflow: hidden;
		white-space: nowrap;

		&:active,
		&:hover {
			transition: var(--background-dark) 0.3s ease;
			background-color: var(--color-background-dark); //$hover-color;

		}

		> div {
			display: flex;
			flex-grow: 1;
			font-size: 1.2em;
			opacity: 0.7;
			white-space: normal;
			padding-right: 4px;
			&.avatar {
				flex-grow: 0;
			}
		}

		> div:nth-last-child(1) {
			justify-content: center;
			flex-grow: 0;
			flex-shrink: 0;
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
	background-repeat: no-repeat;
	background-position: 10px center;
	min-width: 16px;
	min-height: 16px;
	padding: 10px;
	padding-left: 34px;
	text-align: left;
	margin: 0;
}
</style>

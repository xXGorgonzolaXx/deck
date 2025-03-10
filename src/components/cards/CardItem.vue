<!--
  - @copyright Copyright (c) 2018 Julius Härtl <jus@bitgrid.net>
  -
  - @copyright Copyright (c) 2019 Gary Kim <gary@garykim.dev>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  -
  - @author Gary Kim <gary@garykim.dev>
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
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div :class="{'compact': compactMode, 'current-card': currentCard, 'has-labels': card.labels.length > 0, 'is-editing': editing}"
		tag="div"
		class="card"
		@click.self="openCard">
		<div class="card-upper">
			<h3 v-if="showArchived || !canEdit">
				{{ card.title }}
			</h3>
			<h3 v-else-if="!editing" @click.stop="startEditing(card)">
				{{ card.title }}
			</h3>

			<form v-if="editing"
				v-click-outside="cancelEdit"
				class="dragDisabled"
				@keyup.esc="cancelEdit"
				@submit.prevent="finishedEdit(card)">
				<input v-model="copiedCard.title" type="text" autofocus>
				<input type="button" class="icon-confirm" @click="finishedEdit(card)">
			</form>

			<Actions v-if="canEdit && !editing" @click.stop.prevent>
				<ActionButton v-if="showArchived === false" icon="icon-user" @click="assignCardToMe()">
					{{ t('deck', 'Assign to me') }}
				</ActionButton>
				<ActionButton icon="icon-archive" @click="archiveUnarchiveCard()">
					{{ t('deck', (showArchived ? 'Unarchive card' : 'Archive card')) }}
				</ActionButton>
				<ActionButton v-if="showArchived === false" icon="icon-delete" @click="deleteCard()">
					{{ t('deck', 'Delete card') }}
				</ActionButton>
				<ActionButton icon="icon-external" @click.stop="modalShow=true">
					{{ t('deck', 'Move card') }}
				</ActionButton>
				<ActionButton icon="icon-settings-dark" @click="openCard">
					{{ t('deck', 'Card details') }}
				</ActionButton>
			</Actions>

			<Modal v-if="modalShow" title="Move card to another board" @close="modalShow=false">
				<div class="modal__content">
					<Multiselect v-model="selectedBoard"
						:placeholder="t('deck', 'Select a board')"
						:options="boards"
						label="title"
						@select="loadStacksFromBoard" />
					<Multiselect v-model="selectedStack"
						:placeholder="t('deck', 'Select a stack')"
						:options="stacksFromBoard"
						label="title" />

					<button :disabled="!isBoardAndStackChoosen" class="primary" @click="moveCard">
						{{ t('deck', 'Move card') }}
					</button>
					<button @click="modalShow=false">
						{{ t('deck', 'Cancel') }}
					</button>
				</div>
			</Modal>
		</div>
		<ul v-if="card.labels.length > 0" class="labels" @click="openCard">
			<li v-for="label in card.labels" :key="label.id" :style="labelStyle(label)">
				<span>{{ label.title }}</span>
			</li>
		</ul>
		<div v-show="!compactMode" class="card-controls compact-item" @click="openCard">
			<CardBadges :id="id" />
		</div>
	</div>
</template>

<script>
import { Modal, Actions, ActionButton, Multiselect } from '@nextcloud/vue'
import ClickOutside from 'vue-click-outside'
import { mapState, mapGetters } from 'vuex'
import axios from '@nextcloud/axios'

import CardBadges from './CardBadges'
import Color from '../../mixins/color'
import labelStyle from '../../mixins/labelStyle'

export default {
	name: 'CardItem',
	components: { Modal, CardBadges, Actions, ActionButton, Multiselect },
	directives: {
		ClickOutside,
	},
	mixins: [Color, labelStyle],
	props: {
		id: {
			type: Number,
			default: null,
		},
	},
	data() {
		return {
			menuOpened: false,
			editing: false,
			copiedCard: '',
			modalShow: false,
			selectedBoard: '',
			selectedStack: '',
			stacksFromBoard: [],
		}
	},
	computed: {
		...mapState({
			compactMode: state => state.compactMode,
			showArchived: state => state.showArchived,
			currentBoard: state => state.currentBoard,
		}),
		...mapGetters([
			'canEdit',
		]),
		card() {
			return this.$store.getters.cardById(this.id)
		},
		boards() {
			return this.$store.getters.boards.filter(board => {
				return board.id !== this.currentBoard.id
			})
		},
		menu() {
			return []
		},
		currentCard() {
			return this.$route.params.cardId === this.id
		},
		isBoardAndStackChoosen() {
			if (this.selectedBoard === '' || this.selectedStack === '') {
				return false
			}
			return true
		},
	},
	methods: {
		openCard() {
			this.$router.push({ name: 'card', params: { cardId: this.id } })
		},
		togglePopoverMenu() {
			this.menuOpened = !this.menuOpened
		},
		hidePopoverMenu() {
			this.menuOpened = false
		},
		startEditing(card) {
			this.copiedCard = Object.assign({}, card)
			this.editing = true
		},
		finishedEdit(card) {
			if (this.copiedCard.title !== card.title) {
				this.$store.dispatch('updateCardTitle', this.copiedCard)
			}
			this.editing = false
		},
		cancelEdit() {
			this.editing = false
		},
		deleteCard() {
			this.$store.dispatch('deleteCard', this.card)
		},
		archiveUnarchiveCard() {
			this.copiedCard = Object.assign({}, this.card)
			this.copiedCard.archived = !this.copiedCard.archived
			this.$store.dispatch('archiveUnarchiveCard', this.copiedCard)
		},
		assignCardToMe() {
			this.copiedCard = Object.assign({}, this.card)
			this.copiedCard.newUserUid = this.card.owner.uid
			this.$store.dispatch('assignCardToUser', this.copiedCard)
		},
		async loadStacksFromBoard(board) {
			try {
				const url = OC.generateUrl('/apps/deck/stacks/' + board.id)
				const response = await axios.get(url)
				this.stacksFromBoard = response.data
			} catch (err) {
				return err
			}
		},
		moveCard() {
			this.copiedCard = Object.assign({}, this.card)
			this.copiedCard.stackId = this.selectedStack.id
			this.$store.dispatch('moveCard', this.copiedCard)
			this.modalShow = false
		},
	},
}
</script>

<style lang="scss" scoped>
	$card-spacing: 10px;
	$card-padding: 10px;

	.fade-enter-active, .fade-leave-active {
		transition: opacity .125s;
	}
	.fade-enter, .fade-leave-to {
		opacity: 0;
	}

	body.dark .card {
		border: 1px solid var(--color-border);
	}

	.card {
		box-shadow: 0 0 2px 0 var(--color-box-shadow);
		border-radius: 3px;
		font-size: 100%;
		background-color: var(--color-main-background);
		margin-bottom: $card-spacing;

		&.current-card {
			box-shadow: 0 0 5px 0 var(--color-box-shadow);
		}

		.card-upper {
			display: flex;
			min-height: 50px;
			form {
				display: flex;
				padding: 5px 7px;
				width: 100%;
				input[type=text] {
					flex-grow: 1;
				}

			}
			h3 {
				margin: 14px $card-padding;
				flex-grow: 1;
				font-size: 100%;
				cursor: text;
				overflow-x: hidden;
				word-wrap: break-word;
			}
		}

		.labels {
			flex-grow: 1;
			flex-shrink: 1;
			min-width: 0;
			display: flex;
			flex-direction: row;
			margin-left: $card-padding;
			margin-right: $card-padding;
			margin-top: -5px;

			li {
				flex-grow: 0;
				flex-shrink: 1;
				display: flex;
				flex-direction: row;
				overflow: hidden;
				padding: 1px 3px;
				border-radius: 3px;
				font-size: 85%;
				margin-right: 3px;
				margin-bottom: 3px;

				&:hover {
					overflow: unset;
				}

				span {
					flex-grow: 0;
					flex-shrink: 1;
					min-width: 0;
					overflow: hidden;
					text-overflow: ellipsis;
					white-space: nowrap;
				}
			}
		}

		.card-controls {
			display: flex;
			margin-left: $card-padding;
			margin-right: $card-padding;
			& > div {
				display: flex;
				max-height: 44px;
			}
		}
	}

	.compact {
		min-height: 50px;

		&.has-labels {
			padding-bottom: $card-padding;
		}
		.labels {
			height: 6px;
			margin-top: -7px;
			margin-bottom: 3px;
		}
		.labels li {
			width: 30px;
			height: 6px;
			font-size: 0;
			color: transparent;
		}
	}

	.modal__content {
		width: 25vw;
		min-width: 250px;
		height: 120px;
		text-align: center;
		margin: 20px 20px 60px 20px;

		.multiselect {
			margin-bottom: 10px;
		}
	}

	.modal__content button {
		float: right;
	}
</style>

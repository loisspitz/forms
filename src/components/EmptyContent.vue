<!--
  - @copyright Copyright (c) 2020 John Molakvoæ <skjnldsv@protonmail.com>
  -
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
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div class="emptycontent" role="note">
		<div :class="icon" role="img" />
		<h2><slot /></h2>
		<p v-show="$slots.desc">
			<slot name="desc" />
		</p>
	</div>
</template>

<script>
export default {
	name: 'EmptyContent',

	props: {
		icon: {
			type: String,
			default: 'icon-forms',
		},
	},
}
</script>

<style lang="scss">
.emptycontent {
	margin-top: 20vh;
}

</style>

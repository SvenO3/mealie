<template>
  <div>
    <!-- Recommendation Dialog -->
    <BaseDialog
      v-model="recommendationDialog"
      :title="$t('meal-plan.recommendation')"
      :submit-text="$t('general.create')"
      color="primary"
      :icon="$globals.icons.foods"
      @submit="sendRecommendation"
      @close="resetDialog"
    >
    <v-card-text>
        <v-menu
          v-model="state.pickerMenu"
          :close-on-content-click="false"
          transition="scale-transition"
          offset-y
          max-width="290px"
          min-width="auto"
        >
          <template #activator="{ on, attrs2 }">
            <v-text-field
              v-model="newRecommendation.date"
              :label="$t('general.date')"
              :hint="$t('recipe.date-format-hint-yyyy-mm-dd')"
              persistent-hint
              :prepend-icon="$globals.icons.calendar"
              v-bind="attrs2"
              readonly
              v-on="on"
            />
          </template>
          <v-date-picker
            v-model="newRecommendation.date"
            no-title
            :first-day-of-week="firstDayOfWeek"
            :local="$i18n.locale"
            @input="state.pickerMenu = false"
          />
        </v-menu>
        <v-card-text>
          <v-select
            v-model="newRecommendation.entryType"
            :return-object="false"
            :items="planTypeOptions"
            :label="$t('recipe.entry-type')"
          />
        </v-card-text>
      </v-card-text>
      <div style="display: flex; justify-content: center; align-items: center;">
        <SearchFilter v-if="foods" v-model="selectedFoods" :items="foods" :class="attrs.searchFilter.filterClass">
          <v-icon left>
            {{ $globals.icons.foods }}
          </v-icon>
          {{ $t("general.foods") }}
        </SearchFilter>
      </div>
    </BaseDialog>
    <!-- Create Meal Dialog -->
    <BaseDialog
      v-model="state.dialog"
      :title="$tc(newMeal.existing
        ? 'meal-plan.update-this-meal-plan'
        : 'meal-plan.create-a-new-meal-plan'
      )"
      :submit-text="$tc(newMeal.existing
        ? 'general.update'
        : 'general.create'
      )"
      color="primary"
      :icon="$globals.icons.foods"
      :submit-disabled="isCreateDisabled"
      @submit="
        if (newMeal.existing) {
          actions.updateOne(newMeal);
        } else {
          actions.createOne(newMeal);
        }
        resetDialog();
      "
      @close="resetDialog()"
    >
      <v-card-text>
        <v-menu
          v-model="state.pickerMenu"
          :close-on-content-click="false"
          transition="scale-transition"
          offset-y
          max-width="290px"
          min-width="auto"
        >
          <template #activator="{ on, attrs2 }">
            <v-text-field
              v-model="newMeal.date"
              :label="$t('general.date')"
              :hint="$t('recipe.date-format-hint-yyyy-mm-dd')"
              persistent-hint
              :prepend-icon="$globals.icons.calendar"
              v-bind="attrs2"
              readonly
              v-on="on"
            />
          </template>
          <v-date-picker
            v-model="newMeal.date"
            no-title
            :first-day-of-week="firstDayOfWeek"
            :local="$i18n.locale"
            @input="state.pickerMenu = false"
          />
        </v-menu>
        <v-card-text>
          <v-select
            v-model="newMeal.entryType"
            :return-object="false"
            :items="planTypeOptions"
            :label="$t('recipe.entry-type')"
          />
          <v-autocomplete
            v-if="!dialog.note"
            v-model="newMeal.recipeId"
            :label="$t('meal-plan.meal-recipe')"
            :items="search.data.value"
            :loading="search.loading.value"
            :search-input.sync="search.query.value"
            cache-items
            item-text="name"
            item-value="id"
            :return-object="false"
            :rules="[requiredRule]"
          />
          <template v-else>
            <v-text-field v-model="newMeal.title" :rules="[requiredRule]" :label="$t('meal-plan.meal-title')" />
            <v-textarea v-model="newMeal.text" rows="2" :label="$t('meal-plan.meal-note')" />
          </template>
        </v-card-text>
        <v-card-actions class="my-0 py-0">
          <v-switch v-model="dialog.note" class="mt-n3" :label="$t('meal-plan.note-only')" />
        </v-card-actions>
      </v-card-text>
    </BaseDialog>
    <v-row>
      <v-col
        v-for="(plan, index) in mealplans"
        :key="index"
        cols="12"
        sm="12"
        md="3"
        lg="3"
        xl="2"
        class="col-borders my-1 d-flex flex-column"
      >
        <v-card class="mb-2 border-left-primary rounded-sm pa-2">
          <p class="pl-2 mb-1">
            {{ $d(plan.date, "short") }}
          </p>
        </v-card>
        <draggable
          tag="div"
          handle=".handle"
          delay="250"
          :delay-on-touch-only="true"
          :value="plan.meals"
          group="meals"
          :data-index="index"
          :data-box="plan.date"
          style="min-height: 150px"
          @end="onMoveCallback"
        >
          <v-card
            v-for="mealplan in plan.meals"
            :key="mealplan.id"
            class="my-1"
            :class="{ handle: $vuetify.breakpoint.smAndUp }"
          >
            <v-list-item
              @click="editMeal(mealplan)"
            >
              <v-list-item-avatar :rounded="false">
                <RecipeCardImage
                  v-if="mealplan.recipe"
                  :recipe-id="mealplan.recipe.id"
                  tiny
                  icon-size="25"
                  :slug="mealplan.recipe ? mealplan.recipe.slug : ''"
                >
                </RecipeCardImage>
                <v-icon v-else>
                  {{ $globals.icons.primary }}
                </v-icon>
              </v-list-item-avatar>
              <v-list-item-content>
                <v-list-item-title class="mb-1">
                  {{ mealplan.recipe ? mealplan.recipe.name : mealplan.title }}
                </v-list-item-title>
                <v-list-item-subtitle style="min-height: 16px">
                  {{ mealplan.recipe ? mealplan.recipe.description + " " : mealplan.text }}
                </v-list-item-subtitle>
              </v-list-item-content>
            </v-list-item>
            <v-divider class="mx-2"></v-divider>
            <div class="py-2 px-2 d-flex" style="align-items: center">
              <v-btn small icon :class="{ handle: !$vuetify.breakpoint.smAndUp }">
                <v-icon>
                  {{ $globals.icons.arrowUpDown }}
                </v-icon>
              </v-btn>
              <v-menu offset-y>
                <template #activator="{ on, attrs2 }">
                  <v-chip v-bind="attrs2" label small color="accent" v-on="on" @click.prevent>
                    <v-icon left>
                      {{ $globals.icons.tags }}
                    </v-icon>
                    {{ getEntryTypeText(mealplan.entryType) }}
                  </v-chip>
                </template>
                <v-list>
                  <v-list-item
                    v-for="mealType in planTypeOptions"
                    :key="mealType.value"
                    @click="actions.setType(mealplan, mealType.value)"
                  >
                    <v-list-item-title> {{ mealType.text }} </v-list-item-title>
                  </v-list-item>
                </v-list>
              </v-menu>
              <v-btn class="ml-auto" small icon @click="actions.deleteOne(mealplan.id)">
                <v-icon>{{ $globals.icons.delete }}</v-icon>
              </v-btn>
            </div>
          </v-card>
        </draggable>
        <!-- Day Column Actions -->
        <div class="d-flex justify-end mt-auto">
          <BaseButtonGroup
            :buttons="[
              {
                icon: $globals.icons.diceMultiple,
                text: $tc('meal-plan.random-meal'),
                event: 'random',
                children: [
                  {
                    icon: $globals.icons.diceMultiple,
                    text: $tc('meal-plan.breakfast'),
                    event: 'randomBreakfast',
                  },
                  {
                    icon: $globals.icons.diceMultiple,
                    text: $tc('meal-plan.lunch'),
                    event: 'randomLunch',
                  },
                  {
                    icon: $globals.icons.diceMultiple,
                    text: $tc('meal-plan.dinner'),
                    event: 'randomDinner',
                  },
                ],
              },
              {
                icon: $globals.icons.potSteam,
                text: $tc('meal-plan.recommendation'),
                event: 'recommendation',
              },
              {
                icon: $globals.icons.bowlMixOutline,
                text: $tc('meal-plan.random-side'),
                event: 'randomSide',
              },
              {
                icon: $globals.icons.createAlt,
                text: $tc('general.new'),
                event: 'create',
              },
            ]"
            @recommendation="openRecommendationDialog(plan.date)"
            @create="openDialog(plan.date)"
            @randomBreakfast="randomMeal(plan.date, 'breakfast')"
            @randomLunch="randomMeal(plan.date, 'lunch')"
            @randomDinner="randomMeal(plan.date, 'dinner')"
            @randomSide="randomMeal(plan.date, 'side')"
          />
        </div>
      </v-col>
    </v-row>
  </div>
</template>

<script lang="ts">
import axios from "axios"
import {
  computed,
  defineComponent,
  onMounted,
  ref,
  reactive,
  useContext,
  useRoute,
  watch
} from "@nuxtjs/composition-api";
import { format } from "date-fns";
import { SortableEvent } from "sortablejs";
import draggable from "vuedraggable";
import { MealsByDate } from "./types";
import { useMealplans, usePlanTypeOptions, getEntryTypeText } from "~/composables/use-group-mealplan";
import RecipeCardImage from "~/components/Domain/Recipe/RecipeCardImage.vue";
import { PlanEntryType, UpdatePlanEntry } from "~/lib/api/types/meal-plan";
import { useUserApi } from "~/composables/api";
import { useLoggedInState } from "~/composables/use-logged-in-state";
import { useFoodStore, usePublicFoodStore} from "~/composables/store";
import { IngredientFood} from "~/lib/api/types/recipe";
import SearchFilter from "~/components/Domain/SearchFilter.vue";
import { useRecipeFinderPreferences } from "~/composables/use-users/preferences";
import { useHouseholdSelf } from "~/composables/use-households";
import { useRecipeSearch } from "~/composables/recipes/use-recipe-search";

export default defineComponent({
  components: {
    draggable,
    RecipeCardImage,
    SearchFilter,
  },
  props: {
    mealplans: {
      type: Array as () => MealsByDate[],
      required: true,
    },
    actions: {
      type: Object as () => ReturnType<typeof useMealplans>["actions"],
      required: true,
    },
  },
  setup(props) {
    const api = useUserApi();
    const { $auth, $vuetify } = useContext();
    const { household } = useHouseholdSelf();
    const route = useRoute();
    const useMobile = computed(() => $vuetify.breakpoint.smAndDown);
    const requiredRule = (value: any) => !!value || "Required."

    const groupSlug = computed(() => route.value.params.groupSlug || $auth.user?.groupSlug || "");
    const { isOwnGroup } = useLoggedInState();

    const preferences = useRecipeFinderPreferences();

    const attrs = computed(() => {
      return {
        searchFilter: {
          filterClass: useMobile.value ? "ml-4 mb-2" : "mr-4 mb-2",
        },
      };
    })

    const state = ref({
      dialog: false,
      pickerMenu: null as null | boolean,
    });

    const recommendationDialog = ref(false)

    const firstDayOfWeek = computed(() => {
      return household.value?.preferences?.firstDayOfWeek || 0;
    });

    function onMoveCallback(evt: SortableEvent) {
      const supportedEvents = ["drop", "touchend"];

      // Adapted From https://github.com/SortableJS/Vue.Draggable/issues/1029
      const ogEvent: DragEvent = (evt as any).originalEvent;

      if (ogEvent && ogEvent.type in supportedEvents) {
        // The drop was cancelled, unsure if anything needs to be done?
        console.log("Cancel Move Event");
      } else {
        // A Meal was moved, set the new date value and make an update request and refresh the meals
        const fromMealsByIndex = parseInt(evt.from.getAttribute("data-index") ?? "");
        const toMealsByIndex = parseInt(evt.to.getAttribute("data-index") ?? "");

        if (!isNaN(fromMealsByIndex) && !isNaN(toMealsByIndex)) {
          const mealData = props.mealplans[fromMealsByIndex].meals[evt.oldIndex as number];
          const destDate = props.mealplans[toMealsByIndex].date;

          mealData.date = format(destDate, "yyyy-MM-dd");

          props.actions.updateOne(mealData);
        }
      }
    }

    // =====================================================
    // New Meal Dialog

    const dialog = reactive({
      loading: false,
      error: false,
      note: false,
    });

    watch(dialog, () => {
      if (dialog.note) {
        newMeal.recipeId = undefined;
      }
    });

    const newMeal = reactive({
      date: "",
      title: "",
      text: "",
      recipeId: undefined as string | undefined,
      entryType: "dinner" as PlanEntryType,
      existing: false,
      id: 0,
      groupId: "",
      userId: $auth.user?.id || "",
    });

    const newRecommendation = reactive({
      date: "",
      entryType: "dinner" as PlanEntryType,
    })

    const isCreateDisabled = computed(() => {
      if (dialog.note) {
        return !newMeal.title.trim();
      }
      return !newMeal.recipeId;
    });


    function openDialog(date: Date) {
      newMeal.date = format(date, "yyyy-MM-dd");
      state.value.dialog = true;
    }

    function openRecommendationDialog(date: Date) {
      newRecommendation.date = format(date, "yyyy-MM-dd");
      recommendationDialog.value = true;
    }

    const sendRecommendation = async () => {
      try {
        const response = await axios.post("http://host.docker.internal:3000/recommendation", {
          recommendation: newRecommendation,
          foods: selectedFoods.value,
        });
        console.log("Response:", response.data);
      } catch (error) {
        console.error("Error sending recommendation:", error);
      }
    };

    function editMeal(mealplan: UpdatePlanEntry) {
      const { date, title, text, entryType, recipeId, id, groupId, userId } = mealplan;
      if (!entryType) return;

      newMeal.date = date;
      newMeal.title = title || "";
      newMeal.text = text || "";
      newMeal.recipeId = recipeId || undefined;
      newMeal.entryType = entryType;
      newMeal.existing = true;
      newMeal.id = id;
      newMeal.groupId = groupId;
      newMeal.userId = userId || $auth.user?.id || "";

      state.value.dialog = true;
      dialog.note = !recipeId;
    }

    function resetDialog() {
      newMeal.date = "";
      newMeal.title = "";
      newMeal.text = "";
      newMeal.entryType = "dinner";
      newMeal.recipeId = undefined;
      newMeal.existing = false;
    }

    async function randomMeal(date: Date, type: PlanEntryType) {
      const { data } = await api.mealplans.setRandom({
        date: format(date, "yyyy-MM-dd"),
        entryType: type,
      });

      if (data) {
        props.actions.refreshAll();
      }
    }

    // =====================================================
    // Search

    const search = useRecipeSearch(api);
    const planTypeOptions = usePlanTypeOptions();

    onMounted(async () => {
      await search.trigger();
      await Promise.all([hydrateFoods()]);
    });

    // =====================================================
    // Food

    const foodStore = isOwnGroup.value ? useFoodStore() : usePublicFoodStore(groupSlug.value);
    const selectedFoods = ref<IngredientFood[]>([]);
    function addFood(food: IngredientFood) {
      selectedFoods.value.push(food);
      handleFoodUpdates();
    }
    function removeFood(food: IngredientFood) {
      selectedFoods.value = selectedFoods.value.filter((f) => f.id !== food.id);
      handleFoodUpdates();
    }
    function handleFoodUpdates() {
      selectedFoods.value.sort((a, b) => (a.pluralName || a.name).localeCompare(b.pluralName || b.name));
      preferences.value.foodIds = selectedFoods.value.map((food) => food.id);
    }
    watch(
      () => selectedFoods.value,
      () => {
        handleFoodUpdates();
      },
    )

   async function hydrateFoods() {
      if (!preferences.value.foodIds.length) {
        return;
      }
      if (!foodStore.store.value.length) {
        await foodStore.actions.refresh();
      }

      const foods = preferences.value.foodIds
      .map((foodId) => foodStore.store.value.find((food) => food.id === foodId))
      .filter((food) => !!food);

      selectedFoods.value = foods;
    }

    return {
      state,
      onMoveCallback,
      planTypeOptions,
      getEntryTypeText,
      requiredRule,
      isCreateDisabled,

      // Dialog
      dialog,
      newMeal,
      openDialog,
      openRecommendationDialog,
      editMeal,
      resetDialog,
      randomMeal,

      // Search
      search,
      firstDayOfWeek,

      // Recommendation
      sendRecommendation,
      recommendationDialog,
      newRecommendation,
      useMobile,
      attrs,
      isOwnGroup,
      foods: foodStore.store,
      selectedFoods,
      addFood,
      removeFood,
    };
  },
});
</script>

<template>
  <div class="map-wrapper">
    <GeoFilterView class="filter-overlay" @saveFilter="handleFilterData"></GeoFilterView>
    <Sidebar @toggleDraw="toggleDraw"></Sidebar>
    <div id="map" class="map-container"></div>
  </div>
</template>

<script lang="ts" setup>
import {onMounted, ref} from 'vue';
import {Feature, Map, View} from 'ol';
import {Tile as TileLayer, Vector as VectorLayer} from 'ol/layer';
import {OSM, Vector as VectorSource} from 'ol/source';
import {LineString, Point} from 'ol/geom';
import {Icon, Stroke, Style} from 'ol/style';
import axios from 'axios';
import IconStartPin from '../assets/IconStartPin.png';
import IconEndPin from '../assets/IconEndPin.png';
import GeoFilterView from "@/views/GeoFilterView.vue";
import {useToast} from "vue-toastification";
import {boundingExtent} from "ol/extent";
import { Draw } from 'ol/interaction';
import { Polygon } from 'ol/geom';
import Sidebar from "@/components/SideBar.vue";


const toast = useToast();

let projection = ref("EPSG:4326");
let center = ref([-60.457873, 0.584053]);
let zoom = ref(5);
let map = ref<Map | null>(null);
let pointFeatures = ref<Feature[]>([]);
let routeLine = ref<Feature[]>([]);
let pointFinalStar = ref<Feature[]>([]);

let lineLayer = ref<VectorLayer<VectorSource> | null>(null);


function handleFilterData(filterData: { person: number | null, startDate: string | null, endDate: string | null }) {
  pointFeatures.value = [];
  map.value.removeLayers;
  routeLine.value = [];
  pointFinalStar.value = [];

  let getUrl = `http://localhost:8080/tracker/period/${filterData.person}/${filterData.startDate}T00:00:00.000/${filterData.endDate}T00:00:00.000?page=0`;

  getAllPoints(getUrl).then((points) => {
    if (points.length === 0) {
      toast.info("Nenhum ponto encontrado para o filtro selecionado.");
    } else {
      let pointList = new ref(points);
      makeGeometryPointFromArray(pointList, filterData.person);
      lineLayer.value = makeLineFromPoints(pointFeatures);
      console.log(points);
      map.value.addLayer(lineLayer.value);
      adjustMap();
    }
  });
}

const getAllPoints = async (getPointsUrl: string) => {
  try {
    const response = await axios.get(getPointsUrl);
    return response.data.content;
  } catch (error) {
    console.error(error);
    toast.error("Erro ao buscar pontos. Tente novamente mais tarde.");
  }
};

function makeGeometryPointFromArray(arrayOfGeometryObjects, nameFilter?) {
  if (arrayOfGeometryObjects.length === 0) return [];

  if (nameFilter) {
    const startPoint = new Feature({
      geometry: new Point([arrayOfGeometryObjects.value[0].longitude, arrayOfGeometryObjects.value[0].latitude]),
    });

    startPoint.setStyle(new Style({
      image: new Icon({
        src: IconStartPin,
        scale: 0.7,
        anchor: [0.5, 1],
      }),
    }));

    const endPoint = new Feature({
      geometry: new Point([arrayOfGeometryObjects.value[arrayOfGeometryObjects.value.length - 1].longitude, arrayOfGeometryObjects.value[arrayOfGeometryObjects.value.length - 1].latitude]),
    });

    endPoint.setStyle(new Style({
      image: new Icon({
        src: IconEndPin,
        scale: 0.7,
        anchor: [0.5, 1],
      }),
    }));

    pointFinalStar.value.push(startPoint);
    pointFinalStar.value.push(endPoint);

    center.value = endPoint.getGeometry().getCoordinates();
  }

  arrayOfGeometryObjects.value.forEach((pointObj) => {
    const point = new Feature({
      geometry: new Point([pointObj.longitude, pointObj.latitude]),
    });
    point.setStyle(new Style({
      image: new Icon({
        src: IconEndPin,
        scale: 0.7,
        anchor: [0.5, 1],
      }),
    }));
    pointFeatures.value.push(point);
  });
}

function makeLineFromPoints(featureList) {
  if (featureList.value.length === 0) {
    console.log('Nenhum ponto disponível para criar linhas');
    toast.info("Nenhum ponto disponível para criar linhas.");
    return null;
  }

  const groupedById = featureList.value.reduce((acc, feature) => {
    const idText = feature.get('idText');
    if (!acc[idText]) acc[idText] = [];
    acc[idText].push(feature);
    return acc;
  }, {});

  Object.keys(groupedById).forEach((idText) => {
    const points = groupedById[idText];
    if (points.length >= 2) {
      for (let i = 0; i < points.length - 1; i++) {
        const point1 = points[i];
        const point2 = points[i + 1];

        const lineFeature = new Feature({
          geometry: new LineString([point1.getGeometry().getCoordinates(), point2.getGeometry().getCoordinates()]),
        });
        lineFeature.setStyle(new Style({
          stroke: new Stroke({
            color: '#ec1c24',
            width: 5,
          }),
        }));
        routeLine.value.push(lineFeature);
      }
    }
  });
  return new VectorLayer({
    source: new VectorSource({
      features: routeLine.value,
    })
  })
}

const adjustMap = () => {
  const coordinates = pointFeatures.value.map((pontos) =>
      pontos.getGeometry().getCoordinates()
  );
  const extent = boundingExtent(coordinates);
  if (map.value) {
    map.value
        .getView()
        .fit(extent, {padding: [50, 50, 50, 50], maxZoom: 15});
  }
};

let drawInteraction = ref<Draw | null>(null);

function toggleDraw() {
  console.log('asda')
  drawInteraction.value = !drawInteraction.value;
  drawPolygon();
}

const drawPolygon = () => {
  if (drawInteraction.value) {
    map.value?.removeInteraction(drawInteraction.value);
  }

  drawInteraction.value = new Draw({
    source: new VectorSource(),  // This will hold the polygon features
    type: 'Polygon',
  });

  drawInteraction.value.on('drawend', (event) => {
    const polygon = event.feature.getGeometry() as Polygon;
    console.log('Polygon Coordinates:', polygon.getCoordinates());

    pointFeatures.value.push(event.feature);
  });

  map.value?.addInteraction(drawInteraction.value);
};


const createMap = () => {
  map.value = new Map({
    target: 'map',
    layers: [
      new TileLayer({
        source: new OSM(),
      }),
    ],
    view: new View({
      center: center.value,
      zoom: zoom.value,
      projection: projection.value,
    }),
  });

  const vectorLayer = new VectorLayer({
    source: new VectorSource({
      features: pointFeatures.value,
    }),

  });

  const routeLayer = new VectorLayer({
    source: new VectorSource({
      features: routeLine.value,
    }),
  });

  map.value.addLayer(vectorLayer);
  map.value.addLayer(routeLayer);
}
onMounted(() => {

  createMap()

});
</script>

<style scoped>
.map-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  z-index: 1;
}

.filter-overlay {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
  background-color: white;
  padding: 10px;
  border-radius: 8px;
}
</style>

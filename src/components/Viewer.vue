<template>
  <input
    type="file"
    multiple
    id="selectDicomFile"
    style="margin-bottom: 20px"
    @change="handleFileChange"
  />
  <input
    type="file"
    multiple
    id="selectSegFile"
    style="margin-bottom: 20px"
    @change="handleSegFileChange"
  />

  <div id="content"></div>
</template>

<script>
import {
  RenderingEngine,
  setVolumesForViewports,
  Enums,
  volumeLoader,
} from "@cornerstonejs/core";
import cornerstoneDICOMImageLoader from "@cornerstonejs/dicom-image-loader";
import * as cornerstoneTools from "@cornerstonejs/tools";
import { prefetchMetadataInformation } from "../helpers/convertMultiframeImageIds";
import initDemo from "../helpers/initDemo"; 
import SegmentationReader from "@/monai/SegmentationReader";

export default {
  name: "ViewerComponent",
  data() {
    return {
      toolGroupId: "myToolGroup",
      viewport: null,
      element: null,
      renderingEngine: {},
      viewportId: "CT_ORTHOGRAPHIC",
      numberOfFrames: 0,
      toolGroup: {},
      imageIds: [],
      renderingEngineId: "myRenderingEngine",
      volumeId: "cornerstoneStreamingImageVolume:CT_VOLUME_ID",
      segmentationId: "MY_SEGMENTATION_ID",
      labels: {
        pancreas: 1,
        "pancreatic tumor": 2,
      },
    };
  },
  async beforeCreate() {
    console.log("call beforeCreate");
    await initDemo();
    this.toolGroup = cornerstoneTools.ToolGroupManager.createToolGroup(
      this.toolGroupId
    );
    cornerstoneTools.addTool(cornerstoneTools.StackScrollMouseWheelTool);
    this.toolGroup.addTool(cornerstoneTools.SegmentationDisplayTool.toolName);
    this.toolGroup.setToolEnabled(
      cornerstoneTools.SegmentationDisplayTool.toolName
    );

    this.toolGroup.addTool(cornerstoneTools.StackScrollMouseWheelTool.toolName);

    this.toolGroup.setToolActive(
      cornerstoneTools.StackScrollMouseWheelTool.toolName
    );
    // Instantiate a rendering engine
    this.renderingEngine = new RenderingEngine(this.renderingEngineId);
  },
  mounted() {
    console.log("mounted");
    const content = document.getElementById("content");

    // image div
    const element = document.createElement("div");
    element.oncontextmenu = (e) => e.preventDefault();
    element.id = "cornerstone-element";
    element.style.width = "500px";
    element.style.height = "500px";

    const div = document.createElement("div");
    div.style.display = "flex";
    div.style.flexDirection = "row";

    div.appendChild(element);

    content.appendChild(div);

    this.element = element;
  },
  methods: {
    async newSegmentation() {},
    async addSegmentationsToState(file, labels) {
      const parsedFile = await SegmentationReader.parseNrrdData(file);
      console.log(labels);
      console.log(parsedFile);
      // Create a segmentation of the same resolution as the source data
      // using volumeLoader.createAndCacheDerivedVolume.
      const segmentationVolume = await volumeLoader.createAndCacheDerivedVolume(
        this.volumeId,
        {
          volumeId: this.segmentationId,
        }
      );

      // Add the segmentations to state
      cornerstoneTools.segmentation.addSegmentations([
        {
          segmentationId: this.segmentationId,
          representation: {
            // The type of segmentation
            type: cornerstoneTools.Enums.SegmentationRepresentations.Labelmap,
            // The actual segmentation data, in the case of labelmap this is a
            // reference to the source volume of the segmentation.
            data: {
              volumeId: this.segmentationId,
            },
          },
        },
      ]);

      // Add some data to the segmentations
      this.createSegmentation(segmentationVolume);

      await cornerstoneTools.segmentation.addSegmentationRepresentations(
        this.toolGroupId,
        [
          {
            segmentationId: this.segmentationId,
            type: cornerstoneTools.Enums.SegmentationRepresentations.Labelmap,
          },
        ]
      );
    },
    createSegmentation(segmentationVolume) {
      console.log(segmentationVolume);
      // const numberOfFrames = 30;
      // for (let i = 0; i < numberOfFrames; i++) {
      //   if (slice >= 0 && i !== slice) {
      //     // do only one slice (in case of 3D Volume but 2D result e.g. Deeprow2D)
      //     continue;
      //   }

      //   // no segments in this slice
      //   if (
      //     !labelmaps2D[i] ||
      //     !labelmaps2D[i].segmentsOnLabelmap ||
      //     !labelmaps2D[i].segmentsOnLabelmap.length
      //   ) {
      //     operation = "override";
      //   }

      //   const sliceOffset = slicelengthInBytes * i;
      //   const sliceLength = slicelengthInBytes / 2;

      //   let pixelData = new Uint16Array(buffer, sliceOffset, sliceLength);
      //   let srcPixelData = new Uint16Array(srcBuffer, sliceOffset, sliceLength);

      //   if (operation === "overlap" || operation === "override") {
      //     useSourceBuffer = true;
      //   }

      //   for (let j = 0; j < pixelData.length; j++) {
      //     if (operation === "overlap") {
      //       if (pixelData[j] > 0) {
      //         srcPixelData[j] = pixelData[j] + segmentOffset;
      //       }
      //     } else if (operation === "override") {
      //       if (srcPixelData[j] === segmentIndex) {
      //         srcPixelData[j] = 0;
      //       }
      //       if (pixelData[j] > 0) {
      //         srcPixelData[j] = pixelData[j] + segmentOffset;
      //       }
      //     } else {
      //       if (pixelData[j] > 0) {
      //         pixelData[j] = pixelData[j] + segmentOffset;
      //       }
      //     }
      //   }

      //   pixelData = useSourceBuffer ? srcPixelData : pixelData;
      //   labelmaps2D[i] = { pixelData, segmentsOnLabelmap };
      // }
    },
    async handleSegFileChange(e) {
      try {
        const file = e.target.files[0];
        const reader = new FileReader();
        reader.onloadend = async (event) => {
          await this.addSegmentationsToState(event.target.result, this.labels);
        };
        reader.readAsArrayBuffer(file);
      } catch (err) {
        console.log(err);
      }
    },
    handleFileChange(e) {
      // Create a ORTHOGRAPHIC viewport
      const viewportInput = {
        viewportId: this.viewportId,
        type: Enums.ViewportType.ORTHOGRAPHIC,
        element: this.element,
        defaultOptions: {
          background: [0.2, 0, 0.2],
        },
      };

      this.renderingEngine.enableElement(viewportInput);
      // Get the ORTHOGRAPHIC  viewport that was created
      this.viewport = this.renderingEngine.getViewport(this.viewportId);

      this.toolGroup.addViewport(this.viewportId, this.renderingEngineId);

      const imageIds = [];
      Array.from(e.target.files).forEach((file) => {
        const imageId =
          cornerstoneDICOMImageLoader.wadouri.fileManager.add(file);
        imageIds.push(imageId);
      });
      this.numberOfFrames = imageIds.length;
      this.loadAndViewImage(imageIds);
    },
    async loadAndViewImage(imageIds) {
      this.imageIds = imageIds;
      await prefetchMetadataInformation(imageIds);
      const volume = await volumeLoader.createAndCacheVolume(this.volumeId, {
        imageIds: this.imageIds,
      });
      volume.load();
      setVolumesForViewports(
        this.renderingEngine,
        [{ volumeId: this.volumeId }],
        [this.viewportId]
      );

      this.renderingEngine.renderViewport(this.imageIds);
    },
  },
};
</script>

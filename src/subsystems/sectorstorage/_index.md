---
title: Sector Storage
entries:
  - components
# suppressMenu: true
---

type Piece []byte

struct SectorStorageSubsystem {
       SectorSealer   SectorSealer,
       SectorStore    SectorStore,
       SectorBuilders { SectorConfig : SectorBuilder },

       ProverId       UInt

       AddPiece(
         piece Piece,
		 sectorConfig SectorConfig,
       ) error | bool {
           sectorbuilder := SectorBuilders[sectorConfig];
       	   piecePath := SectorStore.WritePiece(piece);
	   response := sectorBuilder.addPiece(PiecePath);

	   MaybeSeal(response.StagedSector, response.BytesRemaining);
       }

       MaybeSeal(StagedSector StagedSector, BytesRmaining Uint);
       
       Seal(stagedSector StagedSector) {
       	    sealedPath := SectorStore.AllocateSealedSector(stagedSector.SectorSize);
			response := SectorSealer.seal(stagedSector, sealedPath, ProverId);
			SectorStore.RegisterMetadata(
			  SectorMetadata {
			    response.CommR,
				response.PersistentAux,
				response.PartialMerkleTreePath,
			  });
			  
       RetrievePiece(CommD Commitment, Start UInt, Length UInt) Piece {
       }
}

// Indexed by CommR
type SectorMetadata struct {
	CommR                   Commitment,
	PersistentAux           SectorPersistentAux,
	MerkleTreePath          Path,
}